<script setup lang="ts">
import { computed } from 'vue'
import type { NestState, RepairPriority } from '@/types/game'
import { NEST_DAMAGE_NAMES, NEST_DAMAGE_EMOJI, REPAIR_PRIORITY_NAMES, REPAIR_PRIORITY_COLORS } from '@/utils/constants'

const props = defineProps<{
  nest: NestState
  foodStock: number
}>()

const emit = defineEmits<{
  (e: 'setPriority', taskId: string, priority: RepairPriority): void
  (e: 'startRepair', taskId: string): void
}>()

const activeTasks = computed(() => {
  return [...props.nest.repairTasks]
    .filter(t => !t.completed)
    .sort((a, b) => {
      const priorityOrder: Record<RepairPriority, number> = { urgent: 0, high: 1, medium: 2, low: 3 }
      return priorityOrder[a.priority] - priorityOrder[b.priority]
    })
})

const completedTasks = computed(() => {
  return [...props.nest.repairTasks]
    .filter(t => t.completed)
    .sort((a, b) => (b.completedAt || 0) - (a.completedAt || 0))
    .slice(0, 3)
})

const conditionColor = computed(() => {
  const c = props.nest.overallCondition
  if (c >= 80) return 'text-green-400'
  if (c >= 50) return 'text-yellow-400'
  if (c >= 30) return 'text-orange-400'
  return 'text-red-400'
})

const conditionBarColor = computed(() => {
  const c = props.nest.overallCondition
  if (c >= 80) return 'bg-green-500'
  if (c >= 50) return 'bg-yellow-500'
  if (c >= 30) return 'bg-orange-500'
  return 'bg-red-500'
})

const priorities: RepairPriority[] = ['low', 'medium', 'high', 'urgent']

const handlePriorityChange = (taskId: string, priority: RepairPriority) => {
  emit('setPriority', taskId, priority)
}

const handleStartRepair = (taskId: string) => {
  emit('startRepair', taskId)
}
</script>

<template>
  <div class="flex flex-col h-full">
    <div class="font-display text-lg text-amber-300 flex items-center gap-2 px-1 mb-3">
      <span>🏠</span> 鸟巢状态
    </div>

    <div class="glass rounded-xl p-3 mb-3">
      <div class="flex items-center justify-between mb-2">
        <span class="text-white/80 text-sm">整体状况</span>
        <span :class="['font-bold text-sm', conditionColor]">
          {{ nest.overallCondition }}%
        </span>
      </div>
      <div class="w-full h-2 bg-black/30 rounded-full overflow-hidden">
        <div
          :class="['h-full rounded-full transition-all duration-500', conditionBarColor]"
          :style="{ width: `${nest.overallCondition}%` }"
        />
      </div>

      <div class="grid grid-cols-3 gap-2 mt-3 text-xs">
        <div class="text-center">
          <div class="text-white/50">孵化效率</div>
          <div :class="['font-bold', nest.incubationEfficiencyMod > 1 ? 'text-red-400' : 'text-green-400']">
            {{ (1 / nest.incubationEfficiencyMod * 100).toFixed(0) }}%
          </div>
        </div>
        <div class="text-center">
          <div class="text-white/50">喂养效率</div>
          <div :class="['font-bold', nest.feedingEfficiencyMod < 1 ? 'text-red-400' : 'text-green-400']">
            {{ (nest.feedingEfficiencyMod * 100).toFixed(0) }}%
          </div>
        </div>
        <div class="text-center">
          <div class="text-white/50">恐惧加成</div>
          <div :class="['font-bold', nest.fearMod > 1 ? 'text-red-400' : 'text-green-400']">
            {{ (nest.fearMod * 100).toFixed(0) }}%
          </div>
        </div>
      </div>
    </div>

    <div class="flex-1 min-h-0 overflow-y-auto scrollbar-hide">
      <div v-if="activeTasks.length === 0 && completedTasks.length === 0" class="text-center py-8 text-white/40">
        <div class="text-3xl mb-2">✨</div>
        <div class="text-sm">鸟巢完好，无需修复</div>
      </div>

      <div v-if="activeTasks.length > 0" class="space-y-2 mb-4">
        <div class="text-xs text-white/50 px-1">待修复 ({{ activeTasks.length }})</div>
        <div
          v-for="task in activeTasks"
          :key="task.id"
          class="glass rounded-lg p-3 border-l-4 transition-all"
          :class="[
            task.priority === 'urgent' ? 'border-red-500 bg-red-500/10' :
            task.priority === 'high' ? 'border-orange-500 bg-orange-500/10' :
            task.priority === 'medium' ? 'border-yellow-500 bg-yellow-500/10' :
            'border-gray-500 bg-gray-500/10'
          ]"
        >
          <div class="flex items-start justify-between gap-2 mb-2">
            <div class="flex-1">
              <div class="flex items-center gap-1.5">
                <span class="text-base">🔧</span>
                <span class="text-white font-medium text-sm">{{ task.area }}</span>
                <span
                  :class="['px-1.5 py-0.5 rounded text-[10px] text-white', REPAIR_PRIORITY_COLORS[task.priority]]"
                >
                  {{ REPAIR_PRIORITY_NAMES[task.priority] }}
                </span>
              </div>
              <div class="text-white/60 text-xs mt-0.5">{{ task.description }}</div>
            </div>
          </div>

          <div class="flex items-center gap-2 mb-2">
            <div class="flex-1 h-1.5 bg-black/30 rounded-full overflow-hidden">
              <div
                class="h-full bg-gradient-to-r from-amber-500 to-amber-400 rounded-full transition-all duration-300"
                :style="{ width: `${task.progress}%` }"
              />
            </div>
            <span class="text-xs text-white/60 whitespace-nowrap">{{ task.progress.toFixed(0) }}%</span>
          </div>

          <div class="flex items-center justify-between text-xs">
            <div class="flex items-center gap-3">
              <span class="text-white/50">
                🍒 {{ task.consumedFood }}/{{ task.requiredFood }}
              </span>
              <span v-if="!task.startedAt" class="text-yellow-400">⏳ 等待开始</span>
              <span v-else class="text-green-400 animate-pulse">⚙️ 修复中</span>
            </div>

            <div class="flex items-center gap-1">
              <button
                v-if="!task.startedAt"
                class="px-2 py-1 bg-green-600 hover:bg-green-500 text-white rounded text-xs transition-colors"
                :disabled="foodStock < 5"
                :class="{ 'opacity-50 cursor-not-allowed': foodStock < 5 }"
                @click="handleStartRepair(task.id)"
              >
                开始
              </button>

              <select
                class="bg-black/40 text-white text-xs rounded px-1.5 py-1 border border-white/10 focus:outline-none focus:border-amber-400"
                :value="task.priority"
                @change="handlePriorityChange(task.id, ($event.target as HTMLSelectElement).value as RepairPriority)"
              >
                <option v-for="p in priorities" :key="p" :value="p">
                  {{ REPAIR_PRIORITY_NAMES[p] }}
                </option>
              </select>
            </div>
          </div>
        </div>
      </div>

      <div v-if="completedTasks.length > 0" class="space-y-2">
        <div class="text-xs text-white/50 px-1">已完成</div>
        <div
          v-for="task in completedTasks"
          :key="task.id"
          class="glass rounded-lg p-2 opacity-60 border-l-4 border-green-500"
        >
          <div class="flex items-center gap-1.5 text-sm">
            <span>✅</span>
            <span class="text-white">{{ task.area }}</span>
            <span class="text-white/50 text-xs">- {{ task.description }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
