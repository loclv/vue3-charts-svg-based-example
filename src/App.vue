<template>
  <Chart
    :size="{ width: 1200, height: 800 }"
    :data="data"
    :margin="margin"
    :direction="direction"
    :axis="axis"
    :hideY=true
  >

    <template #layers>
      <Grid strokeDasharray="" :hideY=true />
      <Area :dataKeys="['name', 'pl']" type="monotone" :areaStyle="{ fill: 'url(#grad)' }" :hideX=true />
      <Line
        :dataKeys="['name', 'pl']"
        type="monotone"
        :lineStyle="{
          stroke: '#9f7aea'
        }"
      />
      <!-- <Marker :value="1000" label="Mean." color="green" strokeWidth="2" strokeDasharray="6 6" /> -->
      <defs>
        <linearGradient id="grad" gradientTransform="rotate(90)">
          <stop offset="0%" stop-color="#be90ff" stop-opacity="1" />
          <stop offset="100%" stop-color="white" stop-opacity="0.4" />
        </linearGradient>
      </defs>
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
import { Chart, Grid, Line } from 'vue3-charts'
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
  color: #2ecc71
}
</style>
