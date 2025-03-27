<template>
  <VChart ref="chartRef" class="chart" :option="option" :autoresize="true" />
</template>

<script setup lang="ts">
import type { ComposeOption } from 'echarts/core'
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import type { LineSeriesOption } from 'echarts/charts'
import { LineChart } from 'echarts/charts'
import type { GridComponentOption, TooltipComponentOption } from 'echarts/components'
import {
  GridComponent,
  LegendComponent,
  TooltipComponent,
} from 'echarts/components'
import VChart from 'vue-echarts'

const props = defineProps<{
  data: {
    name: string
    data: any[]
    color: string
  }[] | any[]
  format?: (value: number, precision?: number) => string
}>()

use([
  CanvasRenderer,
  LineChart,
  GridComponent,
  TooltipComponent,
  LegendComponent,
])

type EChartsOption = ComposeOption<
  | TooltipComponentOption
  | GridComponentOption
  | LineSeriesOption
>

const chartRef = ref<any>()

watch(() => props.data, () => {
  if (Array.isArray(props.data) && props.data[0]?.name) {
    // 多数据序列模式
    chartRef.value?.setOption({
      series: props.data.map(item => ({
        name: item.name,
        data: item.data,
        type: 'line',
        showSymbol: false,
        lineStyle: {
          color: item.color
        },
        itemStyle: {
          color: item.color
        }
      }))
    })
  } else {
    // 单数据序列模式（保持向后兼容）
    chartRef.value?.setOption({
      series: [{
        data: props.data,
        type: 'line',
        showSymbol: false,
      }]
    })
  }
})

const option: EChartsOption = {
  grid: {
    left: 40,
    right: 20,
    top: 10,
    bottom: 20,
  },
  tooltip: {
    trigger: 'axis',
    formatter: (params: any) => {
      if (Array.isArray(params)) {
        const time = formatTime(params[0].value[0])
        const items = params.map((param: any) => {
          const value = props.format 
            ? props.format(param.value[1], 1) 
            : `${param.value[1]}%`
          return `${param.marker} ${param.seriesName}: ${value}`
        }).join('<br/>')
        return `${time}<br/>${items}`
      }
      return ''
    },
    axisPointer: {
      animation: false,
    },
  },
  legend: {
    show: true,
    top: 0,
    right: 20,
  },
  xAxis: {
    type: 'time',
    splitLine: {
      show: false,
    },
    axisLabel: {
      hideOverlap: true,
    },
  },
  yAxis: {
    type: 'value',
    axisLabel: {
      hideOverlap: true,
      showMaxLabel: true,
      formatter: (value: any) => {
        return props.format ? props.format(value, 1) : `${value}%`
      },
    },
  },
  series: Array.isArray(props.data) && props.data[0]?.name
    ? props.data.map(item => ({
        name: item.name,
        data: item.data,
        type: 'line',
        showSymbol: false,
        lineStyle: {
          color: item.color
        },
        itemStyle: {
          color: item.color
        }
      }))
    : [{
        data: props.data,
        type: 'line',
        showSymbol: false,
      }]
}

function formatTime(time: number) {
  const date = new Date(time)
  const hours = date.getHours().toString().padStart(2, '0')
  const minutes = date.getMinutes().toString().padStart(2, '0')
  const seconds = date.getSeconds().toString().padStart(2, '0')
  return `${hours}:${minutes}:${seconds}`
}
</script>

<style>
.chart {
  width: 100%;
  height: 100px;
}
</style>
