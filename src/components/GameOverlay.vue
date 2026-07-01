<script setup lang="ts">
import { computed } from 'vue'

type GamePhase = 'ready' | 'running' | 'paused' | 'levelup' | 'gameover'

const props = defineProps<{
  phase: GamePhase
  level: number
  score: number
  highScore: number
  lastClearedLevel: number
}>()

const emit = defineEmits(['start', 'resume', 'restart', 'continue'])

const overlayConfig = computed(() => {
  switch (props.phase) {
    case 'paused':
      return {
        eyebrow: '系统暂停',
        title: '战术重整中',
        description: '敌军轨道已冻结，准备完成后可立即返回火线。',
        primaryText: '继续战斗',
        secondaryText: '重新开局',
        primaryEvent: 'resume' as const,
        secondaryEvent: 'restart' as const,
      }
    case 'levelup':
      return {
        eyebrow: `第 ${props.lastClearedLevel} 关完成`,
        title: `跃迁至第 ${props.level} 关`,
        description: '敌军加速突入，战场色调已切换，精英怪将更频繁出现。',
        primaryText: '进入下一关',
        secondaryText: '重新开局',
        primaryEvent: 'continue' as const,
        secondaryEvent: 'restart' as const,
      }
    case 'gameover':
      return {
        eyebrow: 'Game Over',
        title: '舰体严重受损',
        description: '本次任务结束，重新集结后再次出击。',
        primaryText: '再次挑战',
        secondaryText: '返回待命',
        primaryEvent: 'restart' as const,
        secondaryEvent: 'start' as const,
      }
    case 'ready':
    default:
      return {
        eyebrow: '星际任务启动',
        title: '突击舰队已待命',
        description: '清除坠落外星战机，守住三层护盾，并不断推进更高关卡。',
        primaryText: '开始战斗',
        secondaryText: '查看操作',
        primaryEvent: 'start' as const,
        secondaryEvent: null,
      }
  }
})

const onPrimary = () => emit(overlayConfig.value.primaryEvent)

const onSecondary = () => {
  const eventName = overlayConfig.value.secondaryEvent
  if (!eventName || props.phase === 'ready') {
    return
  }

  emit(eventName)
}
</script>

<template>
  <div class="absolute inset-0 z-20 flex items-center justify-center bg-slate-950/72 px-5 backdrop-blur-md">
    <div class="panel-glow w-full rounded-[28px] border border-cyan-300/15 p-6 text-center shadow-panel">
      <p class="text-xs uppercase tracking-[0.35em] text-cyan-300/70">
        {{ overlayConfig.eyebrow }}
      </p>
      <h2 class="mt-4 text-3xl font-black text-white">
        {{ overlayConfig.title }}
      </h2>
      <p class="mt-3 text-sm leading-7 text-slate-300">
        {{ overlayConfig.description }}
      </p>

      <div class="mt-6 grid grid-cols-2 gap-3 text-left">
        <div class="rounded-2xl border border-cyan-400/15 bg-slate-900/55 p-4">
          <p class="text-xs uppercase tracking-[0.25em] text-slate-400">分数</p>
          <p class="mt-2 text-2xl font-black text-cyan-300">{{ score }}</p>
        </div>
        <div class="rounded-2xl border border-fuchsia-400/15 bg-slate-900/55 p-4">
          <p class="text-xs uppercase tracking-[0.25em] text-slate-400">最高分</p>
          <p class="mt-2 text-2xl font-black text-fuchsia-300">{{ highScore }}</p>
        </div>
      </div>

      <div
        v-if="phase === 'ready'"
        class="mt-6 rounded-2xl border border-white/10 bg-slate-900/45 p-4 text-left text-sm leading-7 text-slate-300"
      >
        <p>方向键：移动飞船</p>
        <p>空格键：快速射击</p>
        <p>左下摇杆：手机移动</p>
        <p>右下火控：手机开火</p>
      </div>

      <div class="mt-6 flex gap-3">
        <button
          class="flex-1 rounded-2xl bg-gradient-to-r from-cyan-400 to-blue-500 px-5 py-3 text-sm font-bold text-slate-950 transition hover:brightness-110"
          @click="onPrimary"
        >
          {{ overlayConfig.primaryText }}
        </button>
        <button
          class="rounded-2xl border border-white/10 bg-slate-900/70 px-5 py-3 text-sm font-semibold text-slate-100 transition hover:border-cyan-300/30 hover:text-cyan-200"
          :class="{
            'cursor-default opacity-70': phase === 'ready',
          }"
          @click="onSecondary"
        >
          {{ overlayConfig.secondaryText }}
        </button>
      </div>
    </div>
  </div>
</template>
