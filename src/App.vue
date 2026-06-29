<script setup lang="ts">
import { ref, computed, onUnmounted, onMounted } from 'vue'

const API_URL = 'http://localhost:8000'

interface Quote {
  text: string
  source?: string
  length?: number
}

const sampleText = ref('')
const quoteSource = ref('')
const loading = ref(true)
const loadError = ref(false)

const textChars = computed(() => sampleText.value.split(''))
const typed = ref('')
const inputEl = ref<HTMLInputElement | null>(null)
const startTime = ref<number | null>(null)
const finished = ref(false)
const timeLeft = ref(60)
let timer: ReturnType<typeof setInterval> | null = null

async function fetchQuote() {
  loading.value = true
  loadError.value = false
  try {
    const res = await fetch(`${API_URL}/quote`, {
      cache: 'no-store'
    })
    if (!res.ok) throw new Error('Request failed')
    const data: Quote = await res.json()
    sampleText.value = data.text
    quoteSource.value = data.source ?? ''
  } catch (e) {
    loadError.value = true
  } finally {
    loading.value = false
  }
}

function focusInput() {
  inputEl.value?.focus()
}

function onInput() {
  if (!startTime.value) {
    startTime.value = Date.now()
    startTimer()
  }
  if (typed.value.length >= sampleText.value.length) {
    finishTest()
  }
}

function charClass(i: number) {
  if (i >= typed.value.length) return i === typed.value.length ? 'current' : 'untyped'
  return typed.value[i] === sampleText.value[i] ? 'correct' : 'incorrect'
}

function startTimer() {
  timer = setInterval(() => {
    timeLeft.value--
    if (timeLeft.value <= 0) finishTest()
  }, 1000)
}

function finishTest() {
  finished.value = true
  if (timer) clearInterval(timer)
}

const correctChars = computed(() =>
  typed.value.split('').filter((c, i) => c === sampleText.value[i]).length
)

const accuracy = computed(() => {
  if (typed.value.length === 0) return 100
  return Math.round((correctChars.value / typed.value.length) * 100)
})

const wpm = computed(() => {
  if (!startTime.value) return 0
  const minutes = (Date.now() - startTime.value) / 60000
  const words = correctChars.value / 5
  return minutes > 0 ? Math.round(words / minutes) : 0
})

onMounted(fetchQuote)
onUnmounted(() => {
  if (timer) clearInterval(timer)
})

async function reset() {
  typed.value = ''
  startTime.value = null
  finished.value = false
  timeLeft.value = 30
  if (timer) clearInterval(timer)
  await fetchQuote()
  focusInput()
}
</script>

<template>
  <main class="h-screen w-screen p-5">
    <div class="h-full w-full p-10 bg-background-1 flex flex-col rounded-xl">
      <div class="font-mono text-3xl font-extrabold text-tertiary">mood-board</div>

      <div class="flex flex-col justify-center items-center flex-1 gap-8 w-full">
        <div class="flex gap-10 font-mono">
          <div class="flex flex-col items-center">
            <span class="text-5xl font-extrabold text-highlight">{{ wpm }}</span>
            <span class="text-sm text-tertiary uppercase tracking-wide">wpm</span>
          </div>
          <div class="flex flex-col items-center">
            <span class="text-5xl font-extrabold text-highlight">{{ accuracy }}%</span>
            <span class="text-sm text-tertiary uppercase tracking-wide">accuracy</span>
          </div>
          <div class="flex flex-col items-center">
            <div class="flex gap-3">
              <span class="text-5xl font-extrabold text-highlight">10</span>
              <span class="text-5xl font-extrabold text-highlight">30</span>
              <span class="text-5xl font-extrabold text-highlight">60</span>
            </div>
            <span class="text-sm text-tertiary uppercase tracking-wide">seconds</span>
          </div>
        </div>

        <div v-if="loading" class="font-mono text-2xl text-tertiary animate-pulse">
          loading quote...
        </div>

        <div v-else-if="loadError" class="flex flex-col items-center gap-4 font-mono">
          <span class="text-2xl text-secondary">couldn't reach the server</span>
          <button
            @click="fetchQuote"
            class="font-mono font-bold text-secondary bg-highlight px-6 py-2 rounded-lg"
          >
            retry
          </button>
        </div>

        <template v-else>
          <div
            class="font-mono text-4xl font-bold leading-relaxed max-w-3xl text-center cursor-text"
            @click="focusInput"
          >
            <span
              v-for="(char, i) in textChars"
              :key="i"
              :class="charClass(i)"
            >{{ char }}</span>
          </div>

          <span v-if="quoteSource" class="font-mono text-sm text-tertiary opacity-70">
            — {{ quoteSource }}
          </span>

          <input
            ref="inputEl"
            v-model="typed"
            class="opacity-0 absolute pointer-events-none"
            :disabled="finished"
            @input="onInput"
            autofocus
          />

          <button
            v-if="finished"
            @click="reset"
            class="font-mono font-bold text-secondary bg-highlight px-6 py-2 rounded-lg mt-4"
          >
            try again
          </button>
        </template>
      </div>
    </div>
  </main>
</template>

<style scoped>
.correct {
  color: var(--color-highlight);
}
.incorrect {
  color: var(--color-secondary);
  text-decoration: underline;
}
.current {
  color: var(--color-tertiary);
  border-bottom: 3px solid var(--color-tertiary);
}
.untyped {
  color: var(--color-secondary);
  opacity: 0.45;
}
</style>