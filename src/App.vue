<script setup lang="ts">
import { useLocalStorage, useWindowSize } from '@vueuse/core'
import type { ServerData } from '@/types'
import { CARD_WIDTH, JSON_API, MIN_FETCH_INTERVAL } from '@/config'

const { width: WindowWidth } = useWindowSize()

const settings = useLocalStorage('sstl-settings', {
  layout: 'grid' as 'grid' | 'flex' | 'list',
  compactMode: false,
  showCpuChart: false,
  useMonthlyTraffic: true,
}, {
  mergeDefaults: true,
})

const serverData = ref<{
  updated: number
  servers: ServerData[]
}>()
const loading = ref(true)
const error = ref(false)
const fetching = ref(false)
const showSettingPanel = ref(false)
const latestUpdated = ref(0)
const timer = ref<Worker>()

const serverCardCount = computed(() => {
  return Math.floor(WindowWidth.value / CARD_WIDTH) || 1
})

onMounted(() => {
  document.addEventListener('click', (e) => {
    if (showSettingPanel.value && !(e.target as HTMLElement).closest('.setting-panel'))
      showSettingPanel.value = false
  })
  fetch(JSON_API)
    .then(res => res.json())
    .then((data) => {
      serverData.value = data
      timer.value = new Worker(new URL('./worker/timer.js', import.meta.url))
      timer.value.addEventListener('message', () => {
        fetchData()
      })
      timer.value.postMessage('start')
    })
    .catch(() => {
      error.value = true
    })
    .finally(() => {
      loading.value = false
    })
})

onUnmounted(() => {
  if (timer.value)
    timer.value.postMessage('stop')
})

function fetchData() {
  if (fetching.value)
    return
  if (Date.now() - latestUpdated.value < MIN_FETCH_INTERVAL)
    return
  fetching.value = true
  fetch(JSON_API)
    .then(res => res.json())
    .then((data) => {
      serverData.value = data
      error.value = false
    })
    .catch(() => {
      error.value = true
    })
    .finally(() => {
      fetching.value = false
      latestUpdated.value = Date.now()
    })
}
</script>

<template>
  <div class="absolute right-3 top-4 flex flex-col items-end sm:right-6">
    <button @click.stop="showSettingPanel = !showSettingPanel">
      <IconSettings class="size-6 text-gray-500 transition-colors hover:text-black" />
    </button>
    <Transition name="popup-right">
      <div
        v-show="showSettingPanel"
        class="setting-panel z-50 mt-2 min-w-[200px] rounded-lg border bg-white px-4 py-3 shadow-md"
      >
        <h2 class="text-lg font-bold">
          设置
        </h2>
        <div class="flex flex-col gap-1">
          <SettingItem title="布局模式">
            <div class="h-[30px] overflow-hidden rounded-lg border">
              <button
                class="p-1 text-gray-500 transition-colors hover:bg-gray-200"
                :class="{
                  '!bg-gray-300 !text-black': settings.layout === 'grid',
                }"
                @click="settings.layout = 'grid'"
              >
                <IconLayoutGrid class="size-5" />
              </button>
              <button
                class="p-1 text-gray-500 transition-colors hover:bg-gray-200"
                :class="{
                  '!bg-gray-300 !text-black': settings.layout === 'flex',
                }"
                @click="settings.layout = 'flex'"
              >
                <IconLayoutFlex class="size-5" />
              </button>
              <button
                class="p-1 text-gray-500 transition-colors hover:bg-gray-200"
                :class="{
                  '!bg-gray-300 !text-black': settings.layout === 'list',
                }"
                @click="settings.layout = 'list'"
              >
                <IconLayoutList class="size-5" />
              </button>
            </div>
          </SettingItem>
          <SettingItem title="精简显示">
            <Switch v-model="settings.compactMode" />
          </SettingItem>
          <SettingItem v-show="settings.layout !== 'list'" title="CPU图表">
            <Switch v-model="settings.showCpuChart" />
          </SettingItem>
          <SettingItem title="显示周期流量">
            <Switch v-model="settings.useMonthlyTraffic" />
          </SettingItem>
        </div>
      </div>
    </Transition>
  </div>
  <div v-if="loading" class="mx-auto my-2 w-fit rounded-lg bg-gray-100 px-4 py-2">
    加载中
  </div>
  <div v-if="error" class="mx-auto my-2 w-fit rounded-lg bg-gray-100 px-4 py-2">
    数据加载失败，请尝试刷新页面或检查 ServerStatus 服务端状态
  </div>
  <Transition name="popup-bottom">
    <div
      v-if="serverData"
      :class="{
        'grid gap-x-4 gap-y-3': settings.layout === 'grid',
        'flex flex-wrap gap-x-4 gap-y-3': settings.layout === 'flex',
        'flex flex-col gap-y-3': settings.layout === 'list',
      }"
      :style="{
        gridTemplateColumns: `repeat(${serverCardCount}, minmax(0, 1fr))`,
      }"
    >
      <template v-if="settings.layout === 'grid' || settings.layout === 'flex'">
        <ServerCard
          v-for="server, index in serverData.servers" :key="index"
          :server="server"
          :compact-mode="settings.compactMode"
          :show-cpu-chart="settings.showCpuChart"
          :use-monthly-traffic="settings.useMonthlyTraffic"
          :class="{
            'col-span-1': settings.layout === 'grid',
            'min-w-[300px] flex-1': settings.layout === 'flex',
          }"
        />
      </template>
      <template v-if="settings.layout === 'list'">
        <ServerItem
          v-for="server, index in serverData.servers" :key="index"
          :server="server"
          :compact-mode="settings.compactMode"
          :use-monthly-traffic="settings.useMonthlyTraffic"
          class="col-span-1"
        />
      </template>
    </div>
  </Transition>
  <div class="h-16" />
</template>

<style>
.popup-bottom-enter-active,
.popup-bottom-leave-active {
  transition: all 0.3s ease;
}

.popup-bottom-enter-from,
.popup-bottom-leave-to {
  opacity: 0;
  transform: translateY(10px)
}

.popup-right-enter-active,
.popup-right-leave-active {
  transition: all 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.popup-right-enter-from,
.popup-right-leave-to {
  opacity: 0;
  transform: translate(10px, -10px) scale(0.9);
}
</style>
