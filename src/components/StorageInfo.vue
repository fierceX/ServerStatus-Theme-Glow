<template>
  <div class="storage-info">
    <!-- 普通文件系统 -->
    <div v-if="normalDisks.length" class="storage-section">
      <div class="text-sm font-medium text-gray-600 border-b pb-1">
        文件系统
      </div>
      <div v-for="disk in normalDisks" :key="disk.name" class="flex items-center gap-2">
        <span class="text-sm">
          {{ disk.name }}
          <span class="text-gray-500">({{ disk.mount_point }})</span>
        </span>
        <Progress
          :value="disk.used"
          :max="disk.total"
          class="flex-1"
          :class="{
            'progress-danger': getDiskUsagePercent(disk) >= 90,
            'progress-warning': getDiskUsagePercent(disk) >= 70
          }"
        >
          {{ formatBytes(disk.used) }} / {{ formatBytes(disk.total) }}
        </Progress>
      </div>
    </div>

    <!-- ZFS 存储池 -->
    <div v-if="zfsDisks.length" class="storage-section mt-3">
      <div class="text-sm font-medium text-gray-600 border-b pb-1">
        ZFS 存储池
      </div>
      <div v-for="disk in zfsDisks" :key="disk.name" class="flex items-center gap-2">
        <span class="text-sm text-blue-500 font-medium">
          {{ disk.name.replace('zpool-', '') }}
          <span class="text-gray-500 font-normal">({{ disk.mount_point }})</span>
        </span>
        <Progress
          :value="disk.used"
          :max="disk.total"
          class="flex-1"
          :class="{
            'progress-danger': getDiskUsagePercent(disk) >= 90,
            'progress-warning': getDiskUsagePercent(disk) >= 70,
            'progress-zfs': getDiskUsagePercent(disk) < 70
          }"
        >
          {{ formatBytes(disk.used) }} / {{ formatBytes(disk.total) }}
        </Progress>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { DiskInfo } from '../types'
import { formatBytes } from '../utils'
import Progress from './Progress.vue'

const props = defineProps<{
  disks: DiskInfo[]
}>()

const normalDisks = computed(() => 
  props.disks.filter(disk => 
    !disk.name.startsWith('zpool-') && 
    disk.file_system.toLowerCase() !== 'zfs'
  )
)

const zfsDisks = computed(() => 
  props.disks.filter(disk => 
    disk.name.startsWith('zpool-')
  )
)

function getDiskUsagePercent(disk: DiskInfo) {
  return Math.round((disk.used / disk.total) * 100)
}
</script>

<style scoped>
.storage-section {
  @apply space-y-2;
}

.progress-zfs :deep(.progress-bar) {
  @apply bg-blue-500;
}
</style> 