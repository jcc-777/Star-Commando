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

const showPauseButton = computed(() => phase.value === 'running' || phase.value === 'paused' || phase.value === 'levelup')
const pauseButtonText = computed(() => (phase.value === 'running' ? '暂停' : '继续'))

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

const exitToLanding = () => {
  score.value = 0
  level.value = 1
  lives.value = 3
  eliteChance.value = 0
  lastClearedLevel.value = 0
  finalScore.value = 0
  phase.value = 'ready'
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
    <section
      v-if="phase === 'ready'"
      class="start-scene relative flex h-full cursor-pointer items-center justify-center overflow-hidden px-6 text-center"
      @click="startGame"
    >
      <div class="start-scene__stars" />
      <div class="start-scene__stardust" />
      <div class="start-scene__nebula start-scene__nebula--cyan" />
      <div class="start-scene__nebula start-scene__nebula--emerald" />
      <div class="start-scene__nebula start-scene__nebula--pink" />
      <div class="start-scene__planet" />
      <div class="start-scene__planet-continents" />
      <div class="start-scene__planet-clouds" />
      <div class="start-scene__planet-citylights" />
      <div class="start-scene__planet-atmosphere" />
      <div class="start-scene__ring" />
      <div class="start-scene__scanlight" />
      <div class="start-scene__far-fleet" />
      <div class="start-scene__horizon" />
      <div class="start-scene__debris start-scene__debris--left" />
      <div class="start-scene__debris start-scene__debris--mid" />
      <div class="start-scene__debris start-scene__debris--right" />
      <div class="start-scene__trail" />
      <div class="start-scene__hero-ship">
        <span class="start-scene__hero-wing start-scene__hero-wing--left" />
        <span class="start-scene__hero-wing start-scene__hero-wing--right" />
        <span class="start-scene__hero-body" />
        <span class="start-scene__hero-hardpoint start-scene__hero-hardpoint--left" />
        <span class="start-scene__hero-hardpoint start-scene__hero-hardpoint--right" />
        <span class="start-scene__hero-cockpit" />
        <span class="start-scene__hero-engine start-scene__hero-engine--left" />
        <span class="start-scene__hero-engine start-scene__hero-engine--right" />
        <span class="start-scene__hero-id">AX-17</span>
        <span class="start-scene__hero-core" />
      </div>
      <div class="start-scene__escort start-scene__escort--left" />
      <div class="start-scene__escort start-scene__escort--right" />
      <div class="start-scene__fleet start-scene__fleet--left" />
      <div class="start-scene__fleet start-scene__fleet--right" />

      <div class="relative z-10 mx-auto max-w-5xl">
        <p class="hud-label neon-text-cyan">Interstellar Warfront Initiative</p>
        <h1 class="title-art title-art--hero mt-5">
          <span class="title-art__base" data-text="星际突击队">星际突击队</span>
          <span class="title-art__glow">星际突击队</span>
          <span class="title-art__accent">STAR STRIKE SQUADRON</span>
        </h1>
        <p class="mx-auto mt-8 max-w-3xl text-base leading-8 text-slate-200/88 md:text-lg">
          穿越失序星域，迎击自深空坠落的外星战机群。守住舰体护盾，突破封锁火网，在群星尽头夺回人类航道。
        </p>

        <div class="mt-10 flex flex-wrap items-center justify-center gap-3 text-[11px] uppercase tracking-[0.34em] text-slate-200/80">
          <span class="rounded-full border border-cyan-300/20 bg-cyan-400/10 px-4 py-2">全屏网页战斗</span>
          <span class="rounded-full border border-fuchsia-300/20 bg-fuchsia-400/10 px-4 py-2">Canvas Space Shooter</span>
          <span class="rounded-full border border-amber-300/20 bg-amber-400/10 px-4 py-2">Epic Sci-Fi Intro</span>
        </div>

        <div class="mt-14 inline-flex flex-col items-center gap-4 px-8 py-6">
          <div class="start-scene__pulse" />
          <p class="text-lg font-black uppercase tracking-[0.32em] text-white md:text-2xl">
            点击任意处开始游戏
          </p>
          <p class="text-xs uppercase tracking-[0.36em] text-cyan-200/75">
            Arrow Keys Move / Space Fire / Touch Supported
          </p>
        </div>
      </div>

      <div class="absolute bottom-7 left-1/2 z-10 -translate-x-1/2 text-[10px] uppercase tracking-[0.4em] text-slate-300/60">
        Highest Record {{ highScore }}
      </div>
    </section>

    <section v-else class="relative h-full w-full overflow-hidden bg-slate-950">
      <GameCanvas
        :phase="phase"
        :restart-token="restartToken"
        @stats-change="handleStatsChange"
        @level-clear="handleLevelClear"
        @game-over="handleGameOver"
      />

      <div class="pointer-events-none absolute inset-0 bg-[radial-gradient(circle_at_center,transparent_45%,rgba(2,6,23,0.26)_100%)]" />
      <div class="pointer-events-none absolute inset-0 tactical-grid opacity-[0.12]" />

      <div class="pointer-events-none absolute left-4 top-4 z-10 flex flex-wrap gap-3 md:left-6 md:top-6">
        <div class="hud-chip">
          <span class="hud-chip__label">Score</span>
          <span class="hud-chip__value text-cyan-300">{{ score }}</span>
        </div>
        <div class="hud-chip">
          <span class="hud-chip__label">Level</span>
          <span class="hud-chip__value text-fuchsia-300">{{ level }}</span>
        </div>
        <div class="hud-chip">
          <span class="hud-chip__label">Lives</span>
          <span class="hud-chip__value text-emerald-300">{{ lives }}</span>
        </div>
      </div>

      <div class="absolute right-4 top-4 z-20 flex items-center gap-2 md:right-6 md:top-6">
        <button
          v-if="showPauseButton"
          class="topbar-button"
          @click="phase === 'running' ? pauseGame() : resumeGame()"
        >
          {{ pauseButtonText }}
        </button>
        <button class="topbar-button topbar-button--danger" @click="exitToLanding">退出</button>
      </div>

      <div class="pointer-events-none absolute bottom-4 left-1/2 z-10 -translate-x-1/2 rounded-full border border-cyan-300/10 bg-slate-950/35 px-4 py-2 text-[10px] uppercase tracking-[0.32em] text-slate-200/70 backdrop-blur-md md:bottom-6">
        Arrow Keys Move · Space Fire · 500 Points To Next Gate
      </div>

      <GameOverlay
        v-if="phase !== 'running'"
        :phase="phase"
        :level="level"
        :score="phase === 'gameover' ? finalScore : score"
        :high-score="highScore"
        :last-cleared-level="lastClearedLevel"
        @start="exitToLanding"
        @resume="resumeGame"
        @restart="startGame"
        @continue="resumeGame"
      />
    </section>
  </main>
</template>
