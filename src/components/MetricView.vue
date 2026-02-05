<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as echarts from 'echarts'

const host = '//api.sanxunyuan.top'

interface Metric {
  id: number
  metricName: string
  metricTime: string
  metricValue: number
}

const size = ref<number>(100)
const name = ref<string>("cpuTemp")
const loading = ref(false)
const errorMessage = ref<string | null>(null)
const metrics = ref<Metric[]>([])

const chartEl = ref<HTMLDivElement | null>(null)
let chartInstance: echarts.ECharts | null = null

async function loadMetrics() {
  loading.value = true
  errorMessage.value = null

  try {
    const resp = await fetch(
      `${host}/api/metric/search/by-name?metricName=${name.value}&sort=metricTime,desc&size=${size.value}`,
    )
    if (!resp.ok) {
      throw new Error(`请求失败：${resp.status}`)
    }

    const data = await resp.json()
    const list: Metric[] = data._embedded?.metric ?? []
    metrics.value = list

    updateChart()
  } catch (err: any) {
    console.error(err)
    errorMessage.value = err?.message ?? '加载数据失败'
  } finally {
    loading.value = false
  }
}

function updateChart() {
  if (!chartInstance || metrics.value.length === 0) {
    if (chartInstance) {
      chartInstance.clear()
    }
    return
  }

  const metricTimes = metrics.value.map((m) => m.metricTime)
  const metricValues = metrics.value.map((m) => m.metricValue)

  const option: echarts.EChartsOption = {
    tooltip: {
      trigger: 'axis',
    },
    xAxis: {
      type: 'category',
      data: metricTimes,
    },
    yAxis: {
      type: 'value',
    },
    series: [
      {
        type: 'line',
        data: metricValues,
        smooth: true,
        showSymbol: false,
      },
    ],
  }

  chartInstance.setOption(option)
}

function handleResize() {
  if (chartInstance) {
    chartInstance.resize()
  }
}

onMounted(() => {
  if (chartEl.value) {
    chartInstance = echarts.init(chartEl.value)
  }

  window.addEventListener('resize', handleResize)

  // 初次加载
  loadMetrics()
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  if (chartInstance) {
    chartInstance.dispose()
    chartInstance = null
  }
})
</script>

<template>
  <div class="metric-page">
    <section class="metric-controls">
      <label for="size-input">指标数量</label>
      <input
        id="size-input"
        v-model.number="size"
        type="number"
        min="1"
        class="metric-input"
      />
      <label for="name-input">指标名称</label>
      <input
        id="name-input"
        v-model.trim="name"
        type="text"
        class="metric-input"
      />
      <button class="metric-button" :disabled="loading" @click="loadMetrics">
        {{ loading ? '加载中...' : '显示' }}
      </button>
    </section>

    <section class="metric-chart-section">
      <div ref="chartEl" class="metric-chart"></div>
    </section>

    <section class="metric-table-section">
      <p v-if="errorMessage" class="metric-error">
        {{ errorMessage }}
      </p>
      <table v-else class="metric-table">
        <thead>
          <tr>
            <th>id</th>
            <th>指标名称</th>
            <th>指标时间</th>
            <th>指标值</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in metrics" :key="item.id">
            <td>{{ item.id }}</td>
            <td>{{ item.metricName }}</td>
            <td>{{ item.metricTime }}</td>
            <td>{{ item.metricValue }}</td>
          </tr>
          <tr v-if="!loading && metrics.length === 0">
            <td colspan="4" class="metric-empty">暂无数据</td>
          </tr>
        </tbody>
      </table>
    </section>
  </div>
</template>

<style scoped>
.metric-page {
  padding: 24px;
  width: 100%;
  margin: 0 auto;
  box-sizing: border-box;
}

.metric-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
}

.metric-input {
  width: 120px;
  padding: 4px 8px;
}

.metric-button {
  padding: 6px 16px;
  cursor: pointer;
}

.metric-chart-section {
  width: 80%;
  height: 400px;
  margin: 0 auto 24px;
}

.metric-chart {
  width: 100%;
  height: 100%;
}

.metric-table-section {
  width: 80%;
  margin: 0 auto;
}

.metric-table {
  width: 100%;
  border-collapse: collapse;
}

.metric-table th,
.metric-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.metric-table th {
  background-color: #f5f5f5;
}

.metric-empty {
  text-align: center;
  color: #999;
}

.metric-error {
  color: red;
  margin-bottom: 8px;
}
</style>

