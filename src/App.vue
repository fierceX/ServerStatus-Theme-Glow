<template>
  <div class="absolute right-3 sm:right-6 top-4 flex flex-col items-end">
    <button @click.stop="showSettingPanel = !showSettingPanel">
      <IconSettings class="text-gray-500 hover:text-black w-6 h-6 transition-colors" />
    </button>
    <Transition name="popup-right">
      <div
        v-show="showSettingPanel"
        class="setting-panel border shadow-md rounded-lg px-4 py-3 z-50 bg-white mt-2 min-w-[200px]"
      >
        <h2 class="font-bold text-lg">
          设置
        </h2>
        <div class="flex flex-col gap-1">
          <SettingItem title="布局模式">
            <div class="rounded-lg overflow-hidden border h-[30px]">
              <button
                class="p-1 hover:bg-gray-200 text-gray-500 transition-colors"
                :class="{
                  '!bg-gray-300 !text-black': settings.layout === 'grid',
                }"
                @click="settings.layout = 'grid'"
              >
                <IconLayoutGrid class="w-5 h-5" />
              </button>
              <button
                class="p-1 hover:bg-gray-200 text-gray-500 transition-colors"
                :class="{
                  '!bg-gray-300 !text-black': settings.layout === 'flex',
                }"
                @click="settings.layout = 'flex'"
              >
                <IconLayoutFlex class="w-5 h-5" />
              </button>
              <button
                class="p-1 hover:bg-gray-200 text-gray-500 transition-colors"
                :class="{
                  '!bg-gray-300 !text-black': settings.layout === 'list',
                }"
                @click="settings.layout = 'list'"
              >
                <IconLayoutList class="w-5 h-5" />
              </button>
            </div>
          </SettingItem>
          <SettingItem title="精简显示">
            <Switch v-model="settings.compactMode" />
          </SettingItem>
          <SettingItem title="历史图表">
            <Switch v-model="settings.showCpuChart" />
          </SettingItem>
          <SettingItem v-show="settings.showCpuChart" title="历史查看时间">
            <select v-model="settings.historyTimeRange" @change="handleHistoryTimeRangeChange">
              <option value="10m">
                10分钟
              </option>
              <option value="1h">
                1小时
              </option>
              <option value="8h">
                8小时
              </option>
              <option value="12h">
                12小时
              </option>
              <option value="24h">
                24小时
              </option>
            </select>
          </SettingItem>
        </div>
      </div>
    </Transition>
  </div>
  <div v-if="loading" class="w-fit mx-auto my-2 rounded-lg bg-gray-100 px-4 py-2">
    加载中
  </div>
  <div v-if="error" class="w-fit mx-auto my-2 rounded-lg bg-gray-100 px-4 py-2">
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
          v-for="server, index in serverData.current.servers" :key="index"
          :server="server"
          :server-history="findServerHistory(server.name)"
          :compact-mode="settings.compactMode"
          :show-cpu-chart="settings.showCpuChart"
          :class="{
            'col-span-1': settings.layout === 'grid',
            'flex-1 min-w-[300px]': settings.layout === 'flex',
          }"
        />
      </template>
      <template v-if="settings.layout === 'list'">
        <ServerItem
          v-for="server, index in serverData.current.servers" :key="index"
          :server="server"
          :compact-mode="settings.compactMode"
          class="col-span-1"
        />
      </template>
    </div>
  </Transition>
  <div class="h-16" />
</template>

<script setup lang="ts">
import { useLocalStorage, useWindowSize } from '@vueuse/core'
import type { ServerData } from './types'

const JSON_API = '/json/stats.json'
const CARD_WIDTH = 350
const MIN_FETCH_INTERVAL = 500
const HISTORY_FETCH_INTERVAL = 30 * 60 * 1000 // 添加历史数据的请求间隔：30分钟

const { width: WindowWidth } = useWindowSize()

const settings = useLocalStorage('sstl-settings', {
  layout: 'grid',
  compactMode: false,
  showCpuChart: false,
  cpuChartHistoryKeep: 300,
  historyTimeRange: '10m', // 添加新的设置项，默认10分钟
}, {
  mergeDefaults: true,
})

// 修改数据结构以适应新的API返回格式
const serverData = ref<{
  current: {
    updated: number
    servers: ServerData[]
  },
  servers: any[],
  updated?: number  // 添加可选的 updated 属性
}>()
const loading = ref(true)
const error = ref(false)
const fetching = ref(false)
const showSettingPanel = ref(false)
const latestUpdated = ref(0)
const historyLatestUpdated = ref(0) // 添加历史数据的最后更新时间
const timer = ref<Worker>()

// 添加查找服务器历史数据的函数
const findServerHistory = (serverName: string) => {
  if (!serverData.value?.servers) return null
  return serverData.value.servers.find(s => s.name === serverName)
}

const serverCardCount = computed(() => {
  return Math.floor(WindowWidth.value / CARD_WIDTH) || 1
})

onUnmounted(() => {
  if (timer.value)
    timer.value.postMessage('stop')
})

// 添加处理历史时间范围变化的函数
function handleHistoryTimeRangeChange() {
  // 当时间范围变化时，重新获取数据
  fetchData(true)
}

// 获取开始时间的函数，修改为返回时间戳
function getStartTimeParam() {
  const now = new Date()
  let startTime = new Date(now)
  
  switch (settings.value.historyTimeRange) {
    case '10m':
      // 默认10分钟，不需要添加参数
      return null
    case '1h':
      startTime.setHours(now.getHours() - 1)
      break
    case '8h':
      startTime.setHours(now.getHours() - 8)
      break
    case '12h':
      startTime.setHours(now.getHours() - 12)
      break
    case '24h':
      startTime.setDate(now.getDate() - 1)
      break
    default:
      return null
  }
  
  // 返回时间戳（秒）
  return Math.floor(startTime.getTime() / 1000)
}

// 修改获取数据的函数，使用时间戳参数
function fetchData(forceRefresh = false) {
  if (fetching.value && !forceRefresh)
    return
  
  const isHistoryMode = settings.value.historyTimeRange !== '10m'
  const startTime = getStartTimeParam()
  
  // 检查是否需要获取历史数据
  const needFetchHistory = isHistoryMode && 
    (forceRefresh || Date.now() - historyLatestUpdated.value >= HISTORY_FETCH_INTERVAL)
  
  // 检查是否需要获取实时数据
  const needFetchRealtime = forceRefresh || Date.now() - latestUpdated.value >= MIN_FETCH_INTERVAL
  
  if (!needFetchHistory && !needFetchRealtime)
    return
  
  fetching.value = true
  
  // 构建请求URL
  let url = JSON_API
  
  // 如果是历史模式且需要获取历史数据
  if (isHistoryMode && needFetchHistory) {
    // 获取历史数据（用于图表）
    fetch(`${url}?start_time=${startTime}`)
      .then(res => res.json())
      .then((historyData) => {
        // 如果需要获取实时数据
        if (needFetchRealtime) {
          return fetch(url)
            .then(res => res.json())
            .then((realtimeData) => {
              // 更新数据
              if (historyData.servers && realtimeData.current) {
                serverData.value = {
                  current: realtimeData.current,
                  servers: historyData.servers
                }
              } else {
                serverData.value = realtimeData
              }
              error.value = false
              latestUpdated.value = Date.now()
            })
        } else {
          // 只更新历史数据部分
          if (serverData.value && historyData.servers) {
            serverData.value.servers = historyData.servers
          } else {
            serverData.value = historyData
          }
          error.value = false
        }
        historyLatestUpdated.value = Date.now()
      })
      .catch((err) => {
        error.value = true
      })
      .finally(() => {
        fetching.value = false
      })
  } 
  // 如果只需要获取实时数据
  else if (needFetchRealtime) {
    fetch(url)
      .then(res => res.json())
      .then((data) => {
        if (isHistoryMode && serverData.value?.servers) {
          // 在历史模式下，只更新实时数据部分
          serverData.value.current = data.current
        } else {
          // 避免不必要的重新渲染
          if (forceRefresh || !serverData.value || 
              JSON.stringify(serverData.value) !== JSON.stringify(data)) {
            serverData.value = data
          }
        }
        error.value = false
      })
      .catch((err) => {
        error.value = true
      })
      .finally(() => {
        fetching.value = false
        latestUpdated.value = Date.now()
      })
  } else {
    fetching.value = false
  }
}

// 修改初始数据获取逻辑
onMounted(() => {
  document.addEventListener('click', (e) => {
    if (showSettingPanel.value && !(e.target as HTMLElement).closest('.setting-panel'))
      showSettingPanel.value = false
  })
  
  // 初始获取数据
  const startTime = getStartTimeParam()
  const isHistoryMode = settings.value.historyTimeRange !== '10m'
  
  loading.value = true
  
  if (isHistoryMode && startTime) {
    // 获取历史数据，使用时间戳
    fetch(`${JSON_API}?start_time=${startTime}`)
      .then(res => res.json())
      .then((historyData) => {
        // 同时获取实时数据
        return fetch(JSON_API)
          .then(res => res.json())
          .then((realtimeData) => {
            // 合并数据
            if (historyData.servers && realtimeData.current) {
              serverData.value = {
                current: realtimeData.current,
                servers: historyData.servers,
              }
            } else {
              serverData.value = realtimeData
            }
            
            // 设置初始更新时间
            latestUpdated.value = Date.now()
            historyLatestUpdated.value = Date.now()
            
            // 设置定时器
            setupTimer()
          })
      })
      .catch(() => {
        error.value = true
      })
      .finally(() => {
        loading.value = false
      })
  } else {
    // 默认10分钟，直接获取完整数据
    fetch(JSON_API)
      .then(res => res.json())
      .then((data) => {
        serverData.value = data
        latestUpdated.value = Date.now()
        
        // 设置定时器
        setupTimer()
      })
      .catch(() => {
        error.value = true
      })
      .finally(() => {
        loading.value = false
      })
  }
})

// 添加设置定时器的函数
function setupTimer() {
  timer.value = new Worker(new URL('./worker/timer.js', import.meta.url))
  timer.value.addEventListener('message', () => {
    fetchData()
  })
  timer.value.postMessage('start')
}
</script>

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
