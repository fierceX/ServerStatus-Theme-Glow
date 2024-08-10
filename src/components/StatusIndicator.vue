<script setup lang="ts">
defineProps<{
  status: boolean
}>()
</script>

<template>
  <div
    class="inline-block rounded-full"
    :class="{
      'bg-green-400': status,
      'bg-red-500': !status,
    }"
  />
</template>
