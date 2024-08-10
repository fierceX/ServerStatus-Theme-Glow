<script setup lang="ts">
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
        'bg-gray-400/50': value / max < 0.8,
        'bg-orange-300': value / max >= 0.8 && value / max < 0.9,
        'bg-red-300': value / max >= 0.9,
      }"
      :style="{
        width: `${value / max * 100}%`,
      }"
    >
      <slot />
    </div>
  </div>
</template>
