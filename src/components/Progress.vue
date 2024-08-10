<script setup lang="ts">
import { PROGRESS_ORANGE, PROGRESS_RED } from '@/config'

defineProps<{
  value: number
  max: number
}>()
</script>

<template>
  <div
    class="relative inline-block overflow-hidden rounded-full bg-gray-200 text-sm"
  >
    <div
      class="whitespace-nowrap px-2 text-sm transition-[width] duration-500"
      :class="{
        'bg-gray-400/50': value / max < PROGRESS_ORANGE,
        'bg-orange-300': value / max >= PROGRESS_ORANGE && value / max < PROGRESS_RED,
        'bg-red-300': value / max >= PROGRESS_RED,
      }"
      :style="{
        width: `${value / max * 100}%`,
      }"
    >
      <slot />
    </div>
  </div>
</template>
