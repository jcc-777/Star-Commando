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
  <div class="absolute inset-0 z-20 flex items-center justify-center bg-[#02030acc] px-4 backdrop-blur-xl">
    <div class="cyber-panel relative w-full max-w-[92%] overflow-hidden rounded-[28px] p-5 text-center shadow-[0_0_60px_rgba(8,145,178,0.16)] md:max-w-[84%]">
      <div class="pointer-events-none absolute inset-0 bg-[radial-gradient(circle_at_top,rgba(34,211,238,0.16),transparent_40%),linear-gradient(180deg,transparent,rgba(244,114,182,0.06))]" />
      <div class="relative z-10">
        <div class="mx-auto mb-5 h-1.5 w-20 rounded-full bg-gradient-to-r from-cyan-300 via-sky-400 to-fuchsia-400" />
        <p class="text-xs uppercase tracking-[0.42em] text-cyan-300/70">
        {{ overlayConfig.eyebrow }}
        </p>
        <h2 class="mt-3 text-2xl font-black tracking-[0.06em] text-white md:text-3xl">
          {{ overlayConfig.title }}
        </h2>
        <p class="mx-auto mt-2 max-w-md text-sm leading-6 text-slate-300">
          {{ overlayConfig.description }}
        </p>

        <div class="glass-divider my-6" />

        <div class="mt-5 grid grid-cols-2 gap-2 text-left">
          <div class="cyber-card rounded-[22px] p-4">
            <p class="hud-label">分数</p>
            <p class="mt-3 text-3xl font-black text-cyan-300 neon-number">{{ score }}</p>
            <p class="mt-2 text-xs uppercase tracking-[0.25em] text-cyan-300/60">Mission Log</p>
          </div>
          <div class="cyber-card rounded-[22px] p-4">
            <p class="hud-label">最高分</p>
            <p class="mt-3 text-3xl font-black text-fuchsia-300 neon-number">{{ highScore }}</p>
            <p class="mt-2 text-xs uppercase tracking-[0.25em] text-fuchsia-300/60">Archive Node</p>
          </div>
        </div>

        <div
          v-if="phase === 'ready'"
          class="cyber-card mt-6 rounded-[22px] p-4 text-left text-sm leading-7 text-slate-300"
        >
          <div class="mb-3 flex items-center justify-between">
            <p class="hud-label">Control Briefing</p>
            <span class="text-xs uppercase tracking-[0.28em] text-cyan-300/70">Pilot Ready</span>
          </div>
          <p>方向键：移动飞船</p>
          <p>空格键：快速射击</p>
          <p>左下摇杆：手机移动</p>
          <p>右下火控：手机开火</p>
        </div>

        <div
          v-else
          class="cyber-card mt-6 rounded-[22px] p-4 text-left text-sm leading-7 text-slate-300"
        >
          <div class="mb-3 flex items-center justify-between">
            <p class="hud-label">Combat Status</p>
            <span class="text-xs uppercase tracking-[0.28em] text-fuchsia-300/70">System Relay</span>
          </div>
          <p>当前关卡：第 {{ level }} 关</p>
          <p>上一阶段：第 {{ Math.max(lastClearedLevel, 0) }} 关</p>
          <p>建议保持机动，优先清理高速坠落目标与精英怪。</p>
        </div>

        <div class="mt-5 flex gap-2">
          <button
            class="cyber-button pulse-cta flex-1 rounded-[20px] px-5 py-3.5 text-sm font-black uppercase tracking-[0.28em] transition hover:-translate-y-0.5 hover:brightness-110"
            @click="onPrimary"
          >
            {{ overlayConfig.primaryText }}
          </button>
          <button
            class="cyber-button-alt rounded-[20px] px-4 py-3.5 text-sm font-bold uppercase tracking-[0.2em] text-slate-100 transition hover:border-cyan-300/30 hover:text-cyan-200"
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
  </div>
</template>
