<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from 'vue'
import caseClosedUrl from './assets/case-closed.png'
import caseOpenUrl from './assets/case-open.png'

type GameStatus = 'loading' | 'not_started' | 'active' | 'finished'
type ConnectionStatus = 'connecting' | 'connected' | 'disconnected'

interface GameResponse {
  status: Exclude<GameStatus, 'loading'>
  winner: string | null
  ttl_seconds: number | null
}

interface WinnerEvent {
  winner: string
}

const apiUrl = (import.meta.env.VITE_API_URL ?? 'http://localhost:9881/api/v1/game').replace(/\/$/, '')
const isOverlay = window.location.pathname === '/overlay' || new URLSearchParams(window.location.search).has('overlay')
const secretCode = ref('')
const ttlSeconds = ref(300)
const remainingSeconds = ref<number | null>(null)
const gameStatus = ref<GameStatus>('loading')
const connectionStatus = ref<ConnectionStatus>('connecting')
const winner = ref<string | null>(null)
const showSecret = ref(false)
const isSubmitting = ref(false)
const message = ref('Проверяем состояние игры…')
const hasError = ref(false)
let events: EventSource | null = null
let countdownTimer: number | null = null

const isOpen = computed(() => gameStatus.value === 'finished')
const statusTitle = computed(() => {
  if (gameStatus.value === 'loading') return 'Загрузка игры'
  if (gameStatus.value === 'active') return 'Кейс запечатан'
  if (gameStatus.value === 'finished') return 'Кейс открыт!'
  return 'Кейс ждёт новую игру'
})
const statusHint = computed(() => {
  if (gameStatus.value === 'active') return 'Первый правильный код откроет кейс'
  if (gameStatus.value === 'finished') return 'Победитель найден в Twitch-чате'
  return 'Задайте код и запустите раунд'
})
const formattedTime = computed(() => {
  if (remainingSeconds.value === null) return '--:--'
  const minutes = Math.floor(remainingSeconds.value / 60)
  const seconds = remainingSeconds.value % 60
  return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`
})

function stopCountdown() {
  if (countdownTimer !== null) {
    window.clearInterval(countdownTimer)
    countdownTimer = null
  }
}

function startCountdown(ttl: number | null) {
  stopCountdown()
  remainingSeconds.value = ttl
  if (ttl === null || ttl <= 0) return
  countdownTimer = window.setInterval(() => {
    if (remainingSeconds.value === null || remainingSeconds.value <= 1) {
      remainingSeconds.value = 0
      gameStatus.value = 'not_started'
      winner.value = null
      message.value = 'Время раунда истекло. Можно запустить новый.'
      stopCountdown()
      return
    }
    remainingSeconds.value -= 1
  }, 1000)
}

function applyGame(game: GameResponse) {
  gameStatus.value = game.status
  winner.value = game.winner
  if (game.status === 'active') startCountdown(game.ttl_seconds)
  else {
    stopCountdown()
    remainingSeconds.value = game.ttl_seconds
  }
}

function revealWinner(name: string) {
  stopCountdown()
  gameStatus.value = 'finished'
  winner.value = name
  remainingSeconds.value = null
  hasError.value = false
  message.value = `${name} открыл кейс!`
}

async function loadGame() {
  try {
    const response = await fetch(`${apiUrl}/`)
    if (!response.ok) throw new Error(`API ответил ${response.status}`)
    applyGame((await response.json()) as GameResponse)
    hasError.value = false
    message.value = gameStatus.value === 'active' ? 'Раунд уже идёт — ждём ответ в чате.' : 'Готово к новой игре.'
  } catch (error) {
    gameStatus.value = 'not_started'
    hasError.value = true
    message.value = error instanceof Error ? error.message : 'Не удалось связаться с API'
  }
}

async function startGame() {
  const code = secretCode.value.trim()
  if (!code) {
    hasError.value = true
    message.value = 'Введите секретный код.'
    return
  }
  isSubmitting.value = true
  hasError.value = false
  message.value = 'Запечатываем сундук…'
  try {
    const response = await fetch(`${apiUrl}/`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code, ttl_seconds: ttlSeconds.value }),
    })
    if (!response.ok) throw new Error(`Не удалось запустить игру: ${response.status}`)
    applyGame((await response.json()) as GameResponse)
    secretCode.value = ''
    showSecret.value = false
    message.value = 'Раунд начался. Код уже можно отправлять в Twitch-чат.'
  } catch (error) {
    hasError.value = true
    message.value = error instanceof Error ? error.message : 'Не удалось запустить игру'
  } finally {
    isSubmitting.value = false
  }
}

function connectToEvents() {
  events = new EventSource(`${apiUrl}/events`)
  events.addEventListener('open', () => { connectionStatus.value = 'connected' })
  events.addEventListener('game.won', (event) => {
    const data = JSON.parse((event as MessageEvent<string>).data) as WinnerEvent
    revealWinner(data.winner)
  })
  events.addEventListener('error', () => { connectionStatus.value = 'disconnected' })
}

onMounted(() => {
  document.documentElement.classList.toggle('obs-overlay', isOverlay)
  void loadGame()
  connectToEvents()
})
onUnmounted(() => {
  document.documentElement.classList.remove('obs-overlay')
  stopCountdown()
  events?.close()
})
</script>

<template>
  <main class="game-shell" :class="{ 'overlay-mode': isOverlay }">
    <header v-if="!isOverlay" class="topbar">
      <a class="brand" href="#" aria-label="Twitch Safe Game — главная">
        <span class="brand-mark" aria-hidden="true">T</span>
        <span>Twitch Safe Game</span>
      </a>
      <div class="connection" :class="connectionStatus">
        <span class="connection-dot" aria-hidden="true"></span>
        {{ connectionStatus === 'connected' ? 'События подключены' : 'Переподключение…' }}
      </div>
    </header>

    <section class="stage" aria-live="polite">
      <div v-if="!isOverlay" class="stage-copy">
        <p class="eyebrow">Интерактив для стрима</p>
        <h1>{{ statusTitle }}</h1>
        <p>{{ statusHint }}</p>
      </div>

      <div
        v-if="!isOverlay || gameStatus === 'active' || gameStatus === 'finished'"
        class="chest-scene"
        :class="{ open: isOpen, active: gameStatus === 'active' }"
      >
        <div class="aura" aria-hidden="true"></div>
        <span v-for="spark in 12" :key="spark" class="spark" :style="{ '--spark': spark }" aria-hidden="true"></span>
        <div class="winner-reveal" aria-hidden="true">
          <span>Победитель</span>
          <strong>{{ winner }}</strong>
        </div>
        <div class="case-visual" role="img" :aria-label="isOpen ? 'Открытый игровой кейс' : 'Закрытый игровой кейс'">
          <img class="case-image case-closed" :src="caseClosedUrl" alt="" />
          <img class="case-image case-open" :src="caseOpenUrl" alt="" />
        </div>
        <div class="ground-shadow" aria-hidden="true"></div>
      </div>

      <div v-if="gameStatus === 'active'" class="command-card">
        <span>Напиши в чате</span>
        <strong>!code твой_ответ</strong>
        <time>{{ formattedTime }}</time>
      </div>
    </section>

    <aside v-if="!isOverlay" class="control-panel">
      <div class="panel-heading">
        <div><p class="eyebrow">Панель ведущего</p><h2>Новый раунд</h2></div>
        <span class="round-state" :class="gameStatus">
          {{ gameStatus === 'active' ? 'Идёт игра' : gameStatus === 'finished' ? 'Завершена' : 'Ожидание' }}
        </span>
      </div>

      <form @submit.prevent="startGame">
        <label>
          <span>Секретный код</span>
          <div class="secret-field">
            <input
              v-model="secretCode"
              :type="showSecret ? 'text' : 'password'"
              maxlength="128"
              autocomplete="off"
              placeholder="Введите секретный код"
              :disabled="isSubmitting"
            />
            <button
              type="button"
              class="visibility-toggle"
              :aria-label="showSecret ? 'Скрыть секретный код' : 'Показать секретный код'"
              :title="showSecret ? 'Скрыть код' : 'Показать код'"
              @click="showSecret = !showSecret"
            >
              <svg viewBox="0 0 24 24" aria-hidden="true">
                <path d="M2.5 12s3.5-6 9.5-6 9.5 6 9.5 6-3.5 6-9.5 6-9.5-6-9.5-6Z" />
                <circle cx="12" cy="12" r="2.7" />
                <path v-if="!showSecret" class="eye-slash" d="m4 4 16 16" />
              </svg>
            </button>
          </div>
        </label>
        <label>
          <span>Длительность раунда</span>
          <select v-model.number="ttlSeconds" :disabled="isSubmitting">
            <option :value="60">1 минута</option>
            <option :value="300">5 минут</option>
            <option :value="600">10 минут</option>
            <option :value="1800">30 минут</option>
          </select>
        </label>
        <button type="submit" class="start-button" :disabled="isSubmitting">
          <span>{{ isSubmitting ? 'Запускаем…' : gameStatus === 'active' ? 'Начать заново' : 'Запечатать кейс' }}</span>
          <span aria-hidden="true">→</span>
        </button>
      </form>

      <p class="notice" :class="{ error: hasError }">
        <span aria-hidden="true">{{ hasError ? '!' : 'i' }}</span>{{ message }}
      </p>
    </aside>
  </main>
</template>
