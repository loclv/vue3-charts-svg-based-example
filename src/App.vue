<template>
  <Chart
    :size="{ width: 1200, height: 800 }"
    :data="data"
    :margin="margin"
    :direction="direction"
    :axis="axis"
    :hideY="true"
  >
    <template #layers>
      <Grid strokeDasharray="" :hideY="true" />
      <Area
        :dataKeys="['name', 'pl']"
        type="monotone"
        :areaStyle="{ fill: 'url(#grad)' }"
        :hideX="true"
      />
      <Line
        :dataKeys="['name', 'pl']"
        type="monotone"
        :lineStyle="{
          stroke: '#9f7aea'
        }"
      />
    </template>

    <template #widgets>
      <Tooltip
        borderColor="#48CAE4"
        :config="{
          pl: { color: '#9f7aea' },
          avg: { hide: true },
          inc: { hide: true }
        }"
      />
    </template>
  </Chart>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'
import { Grid, Line } from 'vue3-charts'
import Chart from '@/components/chart/Chart.vue'
import { plByMonth } from './data'

export default defineComponent({
  name: 'LineChart',
  components: { Chart, Grid, Line },
  setup() {
    const data = ref(plByMonth)
    const direction = ref('horizontal')
    const margin = ref({
      left: 0,
      top: 20,
      right: 20,
      bottom: 0
    })

    const axis = ref({
      primary: {
        type: 'band'
      },
      secondary: {
        domain: ['dataMin', 'dataMax + 100'],
        type: 'linear',
        ticks: 8
      }
    })

    return { data, direction, margin, axis }
  }
})
</script>

<style>
#app {
  color: #2ecc71;
}
</style>
