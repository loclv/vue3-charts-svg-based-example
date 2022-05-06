<template>
  <div class="chart" ref="chartEl">
    <svg
      :width="size.width"
      :height="size.height"
      :viewBox="`0 0 ${size.width} ${size.height}`"
      @pointermove="onMouseMove"
      @pointerleave="onMouseOut"
    >
      <g class="layers">
        <slot name="layers" />
      </g>
      <g class="axis">
        <axis
          v-if="!hideX"
          position="bottom"
          ref="axBottomEl"
          :rotate="rotateX"
          :isPrimary="direction === 'horizontal'"
        />
        <axis
          v-if="!hideY"
          position="left"
          ref="axLeftEl"
          :isPrimary="direction === 'vertical'"
        />
      </g>
    </svg>
    <div class="widgets">
      <slot name="widgets" />
    </div>
  </div>
</template>

<script lang="ts">
import {
  defineComponent,
  onMounted,
  onUnmounted,
  provide,
  watch,
  reactive,
  ref,
  computed
} from 'vue'

import { pointer } from 'd3-selection'
// import { Chart } from '@/models'
// import { Chart } from 'vue3-charts'
import { bisectCenter } from 'd3-array'

import Axis from './Axis.vue'

interface Size {
  width: number
  height: number
}

interface Data {
  [key: string]: string | number
}

type Direction = 'horizontal' | 'vertical' | 'circular'
type Domain = [string | number, string | number]
type ScaleType = 'band' | 'linear'

interface Margin {
  top: number
  right: number
  bottom: number
  left: number
}

interface AxisConfig {
  domain: Domain
  type: ScaleType
  format?: (_: string) => string
  ticks?: number
  tickValues?: any[]
  hide?: boolean
  rotate?:boolean
  useConfig?: (ax: any) => void
}

interface ChartAxis {
  primary: AxisConfig
  secondary: AxisConfig
}

interface ChartConfig {
  size: Size
  margin: Margin
  direction: Direction
  axis: ChartAxis
  axisSpace: { x: number, y: number }
  controlHover?: boolean
}

const defaultConfig: ChartConfig = {
  size: { width: 500, height: 400 },
  margin: {
    top: 0,
    right: 0,
    bottom: 0,
    left: 0
  },
  direction: 'horizontal',
  axis: {
    primary: {
      domain: ['dataMin', 'dataMax'],
      type: 'band'
    },
    secondary: {
      domain: ['dataMin', 'dataMax'],
      type: 'linear'
    }
  },
  axisSpace: {
    x: 40,
    y: 20
  }
}

class Chart {
  data: Data[]
  layers: Layer[]
  config: ChartConfig
  scales: Scales
  updates: Ref<number>

  constructor(data: Data[], config: Partial<ChartConfig>) {
    this.data = data
    this.layers = []
    this.config = r.mergeDeepLeft(
      r.pickBy((p) => p !== undefined, config),
      defaultConfig
    ) as any
    this.scales = new Scales(this.config.axis)
    this.updates = ref(0)
  }

  get bandScale() {
    if (this.scales.primary.type === 'band') return this.scales.primary
    if (this.scales.secondary.type === 'band') return this.scales.secondary
    return null
  }

  get canvas(): Canvas {
    const { margin, size } = this.config
    const axisSpace = this.config.axisSpace

    if (this.config.axis.primary.hide) {
      axisSpace.y = 0
    }

    if (this.config.axis.secondary.hide) {
      axisSpace.x = 0
    }

    return {
      x: margin.left + axisSpace.x,
      y: margin.top,
      width: size.width - margin.right,
      height: size.height - margin.bottom - axisSpace.y
    }
  }

  public getStackedData(keys: string[]) {
    return stack()
      .keys(keys)
      .order(stackOrderNone)(this.data as any)
  }

  public changeData(data: Data[]) {
    this.data = data
    this.update('data')
  }

  public changeConfig(config: Partial<ChartConfig>) {
    this.config = r.mergeDeepLeft(
      r.pickBy((p) => p !== undefined, config),
      this.config
    ) as any
    this.scales.changeConfig(this.config.axis)

    this.update('config')
  }

  public addLayer(layer: Layer) {
    this.layers.push(layer)
    this.update('layers')
  }

  public removeLayer(id: string) {
    this.layers = this.layers.filter((l) => l.id !== id)
    this.update('layers')
  }

  public getLayers(type?: LayerType) {
    if (type) {
      return this.layers.filter((l) => l.type === type)
    }
    return this.layers
  }

  public getData(keys: string[]): number[] {
    return keys.reduce((arr, key) => arr.concat(getCol(key, this.data)), [])
  }

  public getKeys(idx: number = -1, type: string | null = null) {
    const keys = this.layers
      .filter((l) => {
        if (type) return l.type === type
        return true
      })
      .map((l) => l.dataKeys)

    if (idx === -1) {
      return keys
    }

    return getCol(idx, keys)
  }

  public getStackedKeys(idx: number = -1, type: string | null = null) {
    const keys = this.layers
      .filter((l) => {
        if (type) return l.type === type && l.props.stacked
        return l.props.stacked
      })
      .map((l) => l.dataKeys)

    if (idx === -1) {
      return keys
    }

    return getCol(idx, keys)
  }

  public update(_: string) {
    const stackedData = this.getStackedData(this.getStackedKeys(1))
    this.scales.updateRange(this.canvas, this.config.direction)
    this.scales.updateDomain(this.data, stackedData, this.getKeys())

    this.updates.value += 1
    // console.log('update from:', from)
  }
}


export default defineComponent({
  name: 'CustomChart',
  components: { Axis },
  props: {
    data: {
      type: Array as () => Data[],
      default: () => []
    },
    margin: {
      type: Object as () => Margin,
      default: () => ({
        top: 0,
        right: 0,
        bottom: 0,
        left: 0
      }),
      required: false
    },
    size: {
      type: Object as () => Size,
      default: () => ({ width: 500, height: 400 }),
      required: false
    },
    direction: {
      type: String as () => Direction,
      default: 'horizontal',
      required: false
    },
    axis: {
      type: Object as () => ChartAxis,
      required: false
    },
    config: {
      type: Object as () => Partial<ChartConfig>,
      default: () => ({})
    }
  },
  setup(props) {
    const chartEl = ref(null)
    const axLeftEl = ref<{ $el: SVGGElement }>()
    const axBottomEl = ref<{ $el: SVGGElement }>()
    const axisSpace = reactive<ChartConfig['axisSpace']>({ x: 40, y: 20 })

    const chart: any = new Chart(props.data, props.config)
    const rotateX = computed(() => props.axis && props.axis.primary && props.axis.primary.rotate)
    const hideX = computed(() => {
      return props.axis && props.axis.primary && props.axis.primary.hide
    })

    const hideY = computed(() => {
      return props.axis && props.axis.secondary && props.axis.secondary.hide
    })

    const mouse = reactive({
      index: -1,
      position: { x: 0, y: 0 },
      hover: false
    })

    provide('chart', chart)
    provide('chartMouse', mouse)

    // Data change
    watch(
      () => props.data,
      () => {
        if (chart) chart.changeData(props.data)
      },
      { immediate: true }
    )

    // Config change
    watch(
      () => props.config,
      () => {
        if (chart) chart.changeConfig(props.config)
      }
    )

    watch(
      () => [{ ...axisSpace }, props.direction, props.size, props.margin, props.axis],
      () => {
        if (chart) {
          chart.changeConfig({
            axisSpace: { ...axisSpace },
            direction: props.direction,
            size: props.size,
            margin: props.margin,
            axis: props.axis
          })
        }
      },
      { immediate: true }
    )

    const resizeObserver = new ResizeObserver(entries => {
      for (const entry of entries) {
        if (axBottomEl.value && entry.target === axBottomEl.value.$el) {
          axisSpace.y = entry.contentRect.height
        } else if (axLeftEl.value && entry.target === axLeftEl.value.$el) {
          axisSpace.x = entry.contentRect.width
        }
      }
    })

    onMounted(() => {
      if (axBottomEl.value) {
        resizeObserver.observe(axBottomEl.value.$el)
      }
      if (axLeftEl.value) {
        resizeObserver.observe(axLeftEl.value.$el)
      }
    })

    onUnmounted(() => {
      resizeObserver.disconnect()
      console.log('unmounted')
    })

    function onMouseOut() {
      if (props.config.controlHover === false) {
        return
      }
      mouse.index = -1
      mouse.hover = false
    }

    function onMouseMove(e: MouseEvent) {
      if (props.config.controlHover === false) {
        return
      }
      mouse.hover = true
      mouse.position = { x: pointer(e)[0], y: pointer(e)[1] }

      const { primary } = chart.scales
      const keys = chart.getKeys(0)
      const vals = chart.getData(keys) as number[]

      if (chart.config.direction === 'horizontal') {
        if (primary.type === 'band') {
          const band = primary.bandwidth()
          const delta = mouse.position.x - chart.canvas.x
          mouse.index = Math.round((delta + band / 2) / band) - 1
        } else {
          mouse.index = bisectCenter(
            vals,
            primary.scale.invert(mouse.position.x)
          )
        }
      } else if (chart.config.direction === 'vertical') {
        if (primary.type === 'band') {
          const band = primary.bandwidth()
          const delta = mouse.position.y - chart.canvas.y
          mouse.index = Math.round((delta + band / 2) / band) - 1
        } else {
          mouse.index = bisectCenter(
            vals,
            primary.scale.invert(mouse.position.y)
          )
        }
      }
    }

    return { axBottomEl, axLeftEl, chartEl, hideX, hideY, rotateX, onMouseMove, onMouseOut }
  }
})
</script>

<style>
.chart {
  position: relative;
}
</style>
