<script setup>
defineProps({
  pipelineFriction: {
    type: Object,
    default: null
  }
})

function formatChange(change) {
  if (change > 0) return `+${change}pp`
  if (change < 0) return `${change}pp`
  return '—'
}
</script>

<template>
  <div v-if="pipelineFriction" class="px-6 pb-6 border-b border-gray-200 dark:border-gray-700">
    <div class="grid gap-6 grid-cols-2">
      <div class="space-y-1">
        <p class="text-sm text-gray-500 dark:text-gray-400">Needs Attention</p>
        <div class="flex items-baseline gap-2">
          <span class="text-3xl font-bold dark:text-gray-100">{{ pipelineFriction.needsAttentionPct }}%</span>
          <span class="text-sm text-gray-500 dark:text-gray-400">{{ formatChange(pipelineFriction.needsAttentionChange) }}</span>
        </div>
        <p class="text-xs text-gray-400 dark:text-gray-500">of AI-touched RFEs</p>
      </div>

      <div class="space-y-1">
        <p class="text-sm text-gray-500 dark:text-gray-400">Feasibility Blocked</p>
        <div class="flex items-baseline gap-2">
          <span class="text-3xl font-bold dark:text-gray-100">{{ pipelineFriction.feasibilityBlockedPct }}%</span>
          <span class="text-sm text-gray-500 dark:text-gray-400">{{ formatChange(pipelineFriction.feasibilityBlockedChange) }}</span>
        </div>
        <p class="text-xs text-gray-400 dark:text-gray-500">of AI-touched RFEs</p>
      </div>
    </div>
  </div>
</template>
