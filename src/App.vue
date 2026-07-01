<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import GameCanvas from './components/GameCanvas.vue'
import GameOverlay from './components/GameOverlay.vue'

type GamePhase = 'ready' | 'running' | 'paused' | 'levelup' | 'gameover'

type StatsPayload = {
  score: number
  level: number
  lives: number
  eliteChance: number
}

const HIGH_SCORE_KEY = 'galaxy-game-high-score'

const phase = ref<GamePhase>('ready')
const restartToken = ref(0)
const score = ref(0)
const level = ref(1)
const lives = ref(3)
const eliteChance = ref(0)
const lastClearedLevel = ref(0)
const finalScore = ref(0)
const highScore = ref(Number(localStorage.getItem(HIGH_SCORE_KEY) ?? '0') || 0)

const progressToNextLevel = computed(() => score.value % 500)
const nextLevelTarget = computed(() => 500 - progressToNextLevel.value)
const progressPercent = computed(() => Math.min((progressToNextLevel.value / 500) * 100, 100))
const threatLevel = computed(() => {
  if (level.value >= 6) return 'Omega'
  if (level.value >= 4) return 'High'
  if (level.value >= 2) return 'Medium'
  return 'Low'
})
const phaseText = computed(() => {
  if (phase.value === 'running') return 'Combat Online'
  if (phase.value === 'paused') return 'Systems Paused'
  if (phase.value === 'levelup') return 'Warp Transition'
  if (phase.value === 'gameover') return 'Hull Breached'
  return 'Awaiting Launch'
})

const syncHighScore = (value: number) => {
  if (value > highScore.value) {
    highScore.value = value
    localStorage.setItem(HIGH_SCORE_KEY, String(value))
  }
}

const startGame = () => {
  score.value = 0
  level.value = 1
  lives.value = 3
  eliteChance.value = 0
  finalScore.value = 0
  lastClearedLevel.value = 0
  restartToken.value += 1
  phase.value = 'running'
}

const pauseGame = () => {
  if (phase.value === 'running') {
    phase.value = 'paused'
  }
}

const resumeGame = () => {
  if (phase.value === 'paused' || phase.value === 'levelup') {
    phase.value = 'running'
  }
}

const handleStatsChange = (payload: StatsPayload) => {
  score.value = payload.score
  level.value = payload.level
  lives.value = payload.lives
  eliteChance.value = payload.eliteChance
  syncHighScore(payload.score)
}

const handleLevelClear = (payload: { clearedLevel: number; nextLevel: number }) => {
  lastClearedLevel.value = payload.clearedLevel
  level.value = payload.nextLevel
  phase.value = 'levelup'
}

const handleGameOver = (payload: { score: number }) => {
  finalScore.value = payload.score
  syncHighScore(payload.score)
  phase.value = 'gameover'
}

watch(phase, (value) => {
  if (value === 'gameover') {
    finalScore.value = score.value
  }
})
</script>

<template>
  <main class="relative h-[100dvh] overflow-hidden text-slate-100">
    <div class="pointer-events-none absolute inset-0 tactical-grid opacity-30" />
    <div class="pointer-events-none absolute inset-x-0 top-0 h-40 bg-gradient-to-b from-cyan-400/8 to-transparent" />
    <div class="pointer-events-none absolute -left-24 top-8 h-56 w-56 rounded-full bg-cyan-400/10 blur-[100px]" />
    <div class="pointer-events-none absolute -right-24 bottom-8 h-56 w-56 rounded-full bg-fuchsia-500/10 blur-[120px]" />

    <div class="relative mx-auto flex h-full max-w-[1500px] flex-col gap-3 px-3 py-3 xl:flex-row xl:items-stretch xl:px-6">
      <section class="cyber-shell shrink-0 rounded-[30px] p-4 xl:flex xl:w-[400px] xl:flex-col">
        <div class="relative z-10 flex h-full flex-col">
          <div class="flex items-start justify-between gap-3">
            <div>
              <p class="hud-label neon-text-cyan">Interstellar Strike Protocol</p>
              <h1 class="title-art mt-2">
                <span class="title-art__base">星际突击队</span>
                <span class="title-art__glow">星际突击队</span>
                <span class="title-art__accent">STAR STRIKE</span>
              </h1>
            </div>

            <div class="cyber-card min-w-[88px] rounded-[20px] px-3 py-2 text-center">
              <p class="hud-label text-center text-fuchsia-200/80">Sector</p>
              <p class="mt-1 text-2xl font-black neon-text-pink">L{{ level }}</p>
            </div>
          </div>

          <div class="mt-2 flex flex-wrap gap-2">
            <span class="rounded-full border border-cyan-300/20 bg-cyan-400/10 px-3 py-1 text-[10px] font-bold uppercase tracking-[0.28em] text-cyan-200">
              {{ phaseText }}
            </span>
            <span class="rounded-full border border-fuchsia-300/20 bg-fuchsia-400/10 px-3 py-1 text-[10px] font-bold uppercase tracking-[0.28em] text-fuchsia-200">
              Threat {{ threatLevel }}
            </span>
          </div>

          <div class="glass-divider my-4" />

          <div class="grid grid-cols-2 gap-2">
            <div class="cyber-card rounded-[20px] p-3">
              <p class="hud-label">Current Score</p>
              <p class="mt-2 text-3xl font-black text-cyan-300 neon-number">{{ score }}</p>
              <p class="mt-1 text-[10px] uppercase tracking-[0.25em] text-cyan-200/65">Mission Data</p>
            </div>
            <div class="cyber-card rounded-[20px] p-3">
              <p class="hud-label">Best Record</p>
              <p class="mt-2 text-3xl font-black text-fuchsia-300 neon-number">{{ highScore }}</p>
              <p class="mt-1 text-[10px] uppercase tracking-[0.25em] text-fuchsia-200/65">Archive</p>
            </div>
            <div class="cyber-card rounded-[20px] p-3">
              <p class="hud-label">Shield Core</p>
              <p class="mt-2 text-3xl font-black text-emerald-300 neon-number">{{ lives }}</p>
              <p class="mt-1 text-[10px] uppercase tracking-[0.25em] text-emerald-200/65">Hull Integrity</p>
            </div>
            <div class="cyber-card rounded-[20px] p-3">
              <p class="hud-label">Elite Signal</p>
              <p class="mt-2 text-3xl font-black text-amber-300 neon-number">
                {{ Math.round(eliteChance * 100) }}%
              </p>
              <p class="mt-1 text-[10px] uppercase tracking-[0.25em] text-amber-200/65">Threat Seed</p>
            </div>
          </div>

          <div class="mt-3 cyber-panel rounded-[24px] p-4">
            <div class="flex items-center justify-between">
              <div>
                <p class="hud-label">Warp Progress</p>
                <p class="mt-1 text-base font-bold text-white">{{ phaseText }}</p>
              </div>
              <div class="text-right">
                <p class="hud-label">Threat</p>
                <p class="mt-1 text-base font-bold" :class="threatLevel === 'Omega' ? 'text-fuchsia-300' : 'text-cyan-300'">
                  {{ threatLevel }}
                </p>
              </div>
            </div>

            <div class="mt-3 overflow-hidden rounded-full border border-cyan-300/10 bg-slate-950/70 p-1">
              <div
                class="h-2.5 rounded-full bg-gradient-to-r from-cyan-300 via-sky-400 to-fuchsia-500 transition-all duration-500"
                :style="{ width: `${progressPercent}%` }"
              />
            </div>

            <div class="mt-2 flex items-center justify-between text-[10px] uppercase tracking-[0.24em] text-slate-400">
              <span>Next Gate</span>
              <span>{{ nextLevelTarget === 500 ? 0 : nextLevelTarget }} pts</span>
            </div>
          </div>

          <div class="mt-3 grid grid-cols-[1fr_auto] gap-2">
            <button
              class="cyber-button pulse-cta relative rounded-[22px] px-6 py-4 text-base font-black uppercase tracking-[0.32em] transition hover:-translate-y-0.5 hover:brightness-110"
              @click="phase === 'ready' || phase === 'gameover' ? startGame() : resumeGame()"
            >
              <span class="relative z-10">{{ phase === 'ready' || phase === 'gameover' ? '开始游戏' : '继续作战' }}</span>
            </button>
            <button
              class="cyber-button-alt rounded-[22px] px-4 py-4 text-sm font-bold uppercase tracking-[0.22em] text-slate-100 transition hover:border-fuchsia-300/30 hover:text-fuchsia-200 disabled:cursor-not-allowed disabled:opacity-35"
              :disabled="phase !== 'running'"
              @click="pauseGame"
            >
              暂停
            </button>
          </div>

          <div class="mt-3 grid gap-2 xl:mt-auto">
            <div class="cyber-card rounded-[20px] p-3">
              <div class="flex flex-wrap items-center gap-2 text-[10px] uppercase tracking-[0.22em] text-slate-300">
                <span class="rounded-full border border-cyan-400/15 bg-cyan-400/5 px-3 py-1">方向键移动</span>
                <span class="rounded-full border border-cyan-400/15 bg-cyan-400/5 px-3 py-1">空格射击</span>
                <span class="rounded-full border border-fuchsia-400/15 bg-fuchsia-400/5 px-3 py-1">左摇杆移动</span>
                <span class="rounded-full border border-fuchsia-400/15 bg-fuchsia-400/5 px-3 py-1">右火控开火</span>
              </div>
            </div>

            <div class="grid grid-cols-3 gap-2 text-center text-[10px] uppercase tracking-[0.22em] text-slate-300">
              <div class="cyber-card rounded-[18px] px-2 py-3">AI Assist</div>
              <div class="cyber-card rounded-[18px] px-2 py-3">Warp Core</div>
              <div class="cyber-card rounded-[18px] px-2 py-3">Shield Link</div>
            </div>
          </div>
        </div>
      </section>

      <section class="flex min-h-0 flex-1 items-stretch justify-center">
        <div class="relative flex h-full min-h-0 w-full max-w-[1020px] flex-col">
          <div class="mb-2 flex flex-wrap items-center justify-between gap-2 px-1">
            <div class="flex items-center gap-3">
              <span class="rounded-full border border-cyan-300/20 bg-cyan-400/10 px-3 py-1.5 text-[10px] font-bold uppercase tracking-[0.28em] text-cyan-200">
                {{ phaseText }}
              </span>
              <span class="rounded-full border border-fuchsia-300/20 bg-fuchsia-400/10 px-3 py-1.5 text-[10px] font-bold uppercase tracking-[0.28em] text-fuchsia-200">
                Alert {{ threatLevel }}
              </span>
            </div>
            <p class="text-[10px] uppercase tracking-[0.28em] text-slate-400">
              Neural HUD / Canvas Combat Interface
            </p>
          </div>

          <div class="flex min-h-0 flex-1 items-center justify-center">
            <div class="frame-shell relative h-full max-h-full w-auto aspect-[9/16] max-w-full rounded-[32px] p-2">
            <span class="corner-mark tl" />
            <span class="corner-mark tr" />
            <span class="corner-mark bl" />
            <span class="corner-mark br" />
            <div class="scan-vignette absolute inset-0 z-10 rounded-[28px]" />

              <div class="relative h-full overflow-hidden rounded-[26px] border border-cyan-300/10 bg-slate-950">
              <GameCanvas
                :phase="phase"
                :restart-token="restartToken"
                @stats-change="handleStatsChange"
                @level-clear="handleLevelClear"
                @game-over="handleGameOver"
              />

              <GameOverlay
                v-if="phase !== 'running'"
                :phase="phase"
                :level="level"
                :score="phase === 'gameover' ? finalScore : score"
                :high-score="highScore"
                :last-cleared-level="lastClearedLevel"
                @start="startGame"
                @resume="resumeGame"
                @restart="startGame"
                @continue="resumeGame"
              />
            </div>
            </div>
          </div>
        </div>
      </section>
    </div>
  </main>
</template>
