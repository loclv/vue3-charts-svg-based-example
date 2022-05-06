<template>
  <g
    v-if="data.length && canvas"
    class="layer-grid"
    stroke="black"
    stroke-opacity="0.15"
    stroke-width="1"
    :stroke-dasharray="strokeDasharray"
  >
    <g class="grid-x" v-if="!hideX">
      <line v-for="(y, i) in xLines" :key="i" :x1="canvas.x" :y1="y" :x2="canvas.width" :y2="y" />
    </g>
    <g class="grid-y" v-if="!hideY">
      <line v-for="(x, i) in yLines" :key="i" :x1="x" :y1="canvas.y" :x2="x" :y2="canvas.height" />
    </g>
    <!-- remove border of Grid -->
    <!-- <g>
      <line :x1="canvas.x" :y1="canvas.y" :x2="canvas.width" :y2="canvas.y" />
      <line :x1="canvas.width - 1" :y1="canvas.y" :x2="canvas.width - 1" :y2="canvas.height" />
    </g> -->
  </g>
</template>

<script lang="ts">
import { useChart } from 'vue3-charts'
import { defineComponent, ref, watch } from 'vue'

interface Data {
  [key: string]: string | number
}

interface Canvas {
  x: number
  y: number
  width: number
  height: number
}

export default defineComponent({
  name: 'CustomGrid',
  props: {
    strokeDasharray: {
      type: String,
      default: () => '3 3'
    },
    hideX: {
      type: Boolean,
      default: false
    },
    hideY: {
      type: Boolean,
      default: false
    },
    center: {
      type: Boolean,
      default: true
    }
  },
  setup(props) {
    const chart = useChart()
    const data = ref<Data[]>([])
    const canvas = ref<Canvas | null>(null)
    const xLines = ref<number[]>([])
    const yLines = ref<number[]>([])

    function updateXLines() {
      const { primary, secondary } = chart.scales
      const current = chart.config.direction === 'horizontal' ? secondary : primary

      if (current.type === 'band' && !props.center) {
        const values = current.map(chart.getData(chart.getKeys(0))).map((x: number) => x + current.bandwidth() / 2)
        values.pop()
        xLines.value = values
      } else {
        const ticks = current.ticks()
        xLines.value = current.map(ticks)
      }
    }

    function updateYLines() {
      const { primary, secondary } = chart.scales
      const current = chart.config.direction === 'horizontal' ? primary : secondary

      if (current.type === 'band' && !props.center) {
        const values = current.map(chart.getData(chart.getKeys(0))).map((x: number) => x + current.bandwidth() / 2)
        values.pop()
        yLines.value = values
      } else {
        const ticks = current.ticks()
        yLines.value = current.map(ticks)
      }
    }

    watch(chart.updates, () => {
      data.value = chart.data
      canvas.value = chart.canvas

      updateXLines()
      updateYLines()
    })

    return { data, canvas, xLines, yLines }
  }
})
</script>
