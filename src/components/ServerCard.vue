<script setup lang="ts">
import { CPU_HISTORY_KEEP_TIME } from '@/config'
import type { ServerData } from '@/types'
import { formatBytes, formatTime, hasLoadData, isCountryFlagEmoji, isOnline, parseLabels } from '@/utils'

const props = defineProps<{
  server: ServerData
  compactMode: boolean
  showCpuChart: boolean
  useMonthlyTraffic: boolean
}>()

const StatusChart = defineAsyncComponent(() => import('@/components/StatusChart.vue'))

let cpuHistoryLastUpdated = 0

const cpuHistory = ref<any[]>([])

const labels = computed(() => parseLabels(props.server.labels))
const noLoadData = computed(() => hasLoadData(props.server))
const networkTraffic = computed(() => {
  return props.useMonthlyTraffic && props.server.last_network_in
    ? {
        in: props.server.network_in - props.server.last_network_in,
        out: props.server.network_out - props.server.last_network_out,
      }
    : {
        in: props.server.network_in,
        out: props.server.network_out,
      }
})

watch(() => props.server, () => {
  if (props.server.latest_ts && props.server.cpu) {
    if (props.server.latest_ts <= cpuHistoryLastUpdated)
      return

    const list = cpuHistory.value.slice()
    list.push({
      name: Date.now(),
      value: [
        props.server.latest_ts * 1000,
        props.server.cpu,
      ],
    })
    while (list[0].name < Date.now() - CPU_HISTORY_KEEP_TIME * 1000)
      list.shift()

    cpuHistory.value = list
    cpuHistoryLastUpdated = props.server.latest_ts
  }
})
</script>

<template>
  <div
    class="relative rounded-xl bg-gray-100 px-4 py-3"
  >
    <div class="group absolute right-4 top-4 flex flex-col items-end">
      <StatusIndicator
        :status="isOnline(server)"
        class="size-3"
      />
      <div class="z-[9999] mt-1 hidden rounded-xl border border-gray-400 bg-gray-100 p-2 text-sm group-hover:block">
        <div class="flex gap-2">
          <div class="flex items-center gap-1">
            IPv4
            <StatusIndicator
              :status="server.online4"
              class="size-2"
            />
          </div>
          <div class="flex items-center gap-1">
            IPv6
            <StatusIndicator
              :status="server.online6"
              class="size-2"
            />
          </div>
        </div>
        <div v-if="server.latest_ts !== undefined">
          最后上报时间<br>
          {{ formatTime(server.latest_ts) }}
        </div>
      </div>
    </div>
    <div class="flex items-center gap-2 text-lg">
      <span v-if="isCountryFlagEmoji(server.location)">
        {{ server.location }}
      </span>
      <img
        v-else
        :src="`/image/flags/${server.location.toLowerCase()}.svg`" :alt="`${server.location} flag`"
        class="inline-block h-4 rounded-sm"
      >
      <img
        v-if="labels.os !== undefined"
        :src="`/image/os/${labels.os}.svg`" :alt="`${labels.os} os`"
        class="inline-block h-4 rounded-sm"
      >
      {{ server.alias || server.name }}
    </div>
    <div>
      运行时间
      <span
        :class="{
          'text-red-500': !isOnline(server),
        }"
      >
        {{ isOnline(server) ? server.uptime : '离线' }}
      </span>
    </div>
    <div v-if="!compactMode" class="flex items-center gap-2">
      负载
      <Bandage v-if="noLoadData">
        无数据
      </Bandage>
      <Bandage v-if="server.load !== undefined">
        {{ server.load }}
      </Bandage>
      <Bandage v-if="server.load_1 !== undefined">
        {{ server.load_1 }}
      </Bandage>
      <Bandage v-if="server.load_5 !== undefined">
        {{ server.load_5 }}
      </Bandage>
      <Bandage v-if="server.load_15 !== undefined">
        {{ server.load_15 }}
      </Bandage>
    </div>
    <div v-if="server.cpu !== undefined" class="flex items-center gap-2">
      CPU
      <Progress
        :value="server.cpu" :max="100"
        :text="`${server.cpu}%`"
        class="flex-1"
      >
        {{ server.cpu }}%
      </Progress>
    </div>
    <StatusChart v-if="showCpuChart" :data="cpuHistory" />
    <div v-if="server.memory_total !== undefined" class="flex items-center gap-2">
      内存
      <Progress
        :value="server.memory_used" :max="server.memory_total"
        class="flex-1"
      >
        {{ formatBytes(server.memory_used * 1024) }} / {{ formatBytes(server.memory_total * 1024) }}
      </Progress>
    </div>
    <div v-if="server.hdd_total !== undefined" class="flex items-center gap-2">
      硬盘
      <Progress
        :value="server.hdd_used" :max="server.hdd_total"
        class="flex-1"
      >
        {{ formatBytes(server.hdd_used * 1024 * 1024) }} / {{ formatBytes(server.hdd_total * 1024 * 1024) }}
      </Progress>
    </div>
    <div v-if="server.network_rx !== undefined" class="flex items-center gap-2">
      网络
      <Bandage class="flex items-center">
        <IconDownload class="size-4" />{{ formatBytes(server.network_rx, 1) }}/s
      </Bandage>
      <Bandage class="flex items-center">
        <IconUpload class="size-4" />{{ formatBytes(server.network_tx, 1) }}/s
      </Bandage>
    </div>
    <div v-if="server.network_in !== undefined && !compactMode" class="flex items-center gap-2">
      流量
      <Bandage class="flex items-center">
        <IconDownload class="size-4" />{{ formatBytes(networkTraffic.in, 1) }}
      </Bandage>
      <Bandage class="flex items-center">
        <IconUpload class="size-4" />{{ formatBytes(networkTraffic.out, 1) }}
      </Bandage>
    </div>
    <div v-if="server.swap_total !== undefined && !compactMode">
      SWAP
      <Bandage>
        {{ formatBytes(server.swap_used * 1024) }} / {{ formatBytes(server.swap_total * 1024) }}
      </Bandage>
    </div>
    <div
      v-if=" !compactMode"
      class="mt-1 flex flex-wrap gap-1"
    >
      <Bandage v-if="server.tcp_count !== undefined">
        TCP {{ server.tcp_count }}
      </Bandage>
      <Bandage v-if="server.udp_count !== undefined">
        UDP {{ server.udp_count }}
      </Bandage>
      <Bandage v-if="server.process_count !== undefined">
        进程 {{ server.process_count }}
      </Bandage>
      <Bandage v-if="server.thread_count !== undefined">
        线程 {{ server.thread_count }}
      </Bandage>
    </div>
  </div>
</template>
