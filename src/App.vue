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
  <main class="min-h-screen overflow-hidden bg-slate-950 text-slate-100">
    <div class="mx-auto flex min-h-screen max-w-7xl flex-col gap-6 px-4 py-6 lg:flex-row lg:px-6">
      <section class="panel-glow flex w-full flex-col rounded-[28px] border border-cyan-400/15 p-5 shadow-panel lg:max-w-sm">
        <div class="flex items-center justify-between">
          <div>
            <p class="text-sm uppercase tracking-[0.35em] text-cyan-300/70">Vue 3 Canvas Shooter</p>
            <h1 class="mt-2 text-3xl font-black tracking-wide text-glow">星际突击队</h1>
          </div>
          <div
            class="animate-float rounded-full border border-fuchsia-400/30 bg-fuchsia-500/10 px-4 py-2 text-xs font-semibold uppercase tracking-[0.3em] text-fuchsia-200"
          >
            Lv {{ level }}
          </div>
        </div>

        <div class="mt-6 grid grid-cols-2 gap-3">
          <div class="rounded-2xl border border-cyan-400/15 bg-slate-900/60 p-4">
            <p class="text-xs uppercase tracking-[0.25em] text-slate-400">当前分数</p>
            <p class="mt-2 text-3xl font-black text-cyan-300">{{ score }}</p>
          </div>
          <div class="rounded-2xl border border-fuchsia-400/15 bg-slate-900/60 p-4">
            <p class="text-xs uppercase tracking-[0.25em] text-slate-400">最高记录</p>
            <p class="mt-2 text-3xl font-black text-fuchsia-300">{{ highScore }}</p>
          </div>
          <div class="rounded-2xl border border-emerald-400/15 bg-slate-900/60 p-4">
            <p class="text-xs uppercase tracking-[0.25em] text-slate-400">生命值</p>
            <p class="mt-2 text-3xl font-black text-emerald-300">{{ lives }}</p>
          </div>
          <div class="rounded-2xl border border-amber-400/15 bg-slate-900/60 p-4">
            <p class="text-xs uppercase tracking-[0.25em] text-slate-400">精英率</p>
            <p class="mt-2 text-3xl font-black text-amber-300">{{ Math.round(eliteChance * 100) }}%</p>
          </div>
        </div>

        <div class="mt-5 rounded-2xl border border-cyan-400/10 bg-slate-900/50 p-4">
          <div class="flex items-center justify-between text-sm text-slate-300">
            <span>距离下一关</span>
            <span>{{ nextLevelTarget === 500 ? 0 : nextLevelTarget }} 分</span>
          </div>
          <div class="mt-3 h-3 overflow-hidden rounded-full bg-slate-800">
            <div
              class="h-full rounded-full bg-gradient-to-r from-cyan-400 via-violet-500 to-fuchsia-500 transition-all duration-300"
              :style="{ width: `${Math.min((progressToNextLevel / 500) * 100, 100)}%` }"
            />
          </div>
        </div>

        <div class="mt-5 flex gap-3">
          <button
            class="flex-1 rounded-2xl bg-gradient-to-r from-cyan-400 to-blue-500 px-4 py-3 text-sm font-bold text-slate-950 transition hover:brightness-110"
            @click="phase === 'ready' || phase === 'gameover' ? startGame() : resumeGame()"
          >
            {{ phase === 'ready' || phase === 'gameover' ? '开始任务' : '继续战斗' }}
          </button>
          <button
            class="rounded-2xl border border-white/10 bg-slate-900/70 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-cyan-300/40 hover:text-cyan-200 disabled:cursor-not-allowed disabled:opacity-40"
            :disabled="phase !== 'running'"
            @click="pauseGame"
          >
            暂停
          </button>
        </div>

        <div class="mt-6 space-y-3 text-sm text-slate-300">
          <div class="rounded-2xl border border-white/5 bg-slate-900/40 p-4">
            <p class="font-semibold text-slate-100">操作说明</p>
            <p class="mt-2 leading-7">
              键盘使用 <span class="text-cyan-300">方向键</span> 控制移动，<span class="text-cyan-300">空格键</span>
              连发。手机端使用左下角摇杆和右下角开火键。
            </p>
          </div>
          <div class="rounded-2xl border border-white/5 bg-slate-900/40 p-4">
            <p class="font-semibold text-slate-100">战场规则</p>
            <p class="mt-2 leading-7">
              每累计 <span class="text-fuchsia-300">500 分</span> 自动进入下一关，敌人速度提升、背景色切换，并开始刷新更高概率的精英怪。
            </p>
          </div>
        </div>
      </section>

      <section class="flex min-h-[70vh] flex-1 items-center justify-center">
        <div class="relative aspect-[9/16] w-full max-w-[460px] overflow-hidden rounded-[32px] border border-cyan-400/20 bg-slate-950 shadow-[0_0_80px_rgba(14,165,233,0.15)]">
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
      </section>
    </div>
  </main>
</template>
