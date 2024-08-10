<script setup lang="ts">
defineProps<{
  title: string
}>()
</script>

<template>
  <div class="flex items-center justify-between">
    {{ title }}
    <slot />
  </div>
</template>
