<script setup lang="ts">
import { ref, computed, watch } from 'vue'

interface Category {
  name: string
  id: string
  description: string
}

interface Layer {
  name: string
  color: string
  categoryValues: Record<string, number>
}

interface RadarSkeleton {
  strokeWidth: number
  strokeColor: string
  strokeOpacity: number
}

interface Props {
  categories: Category[]
  layers: Layer[]
  alignTo: 'top' | 'polarZero'
  stepCount?: number
  radarSkeleton?: RadarSkeleton
  activeLayer: string
}

const props = withDefaults(defineProps<Props>(), {
  stepCount: 5,
  radarSkeleton: () => ({
    strokeWidth: 1,
    strokeColor: '#dfdfdf',
    strokeOpacity: 1,
  }),
})

// ============================================================================
// CONSTANTS & CONFIGURATION
// ============================================================================

const CHART_RADIUS = 100
const CHART_CENTER = { x: CHART_RADIUS, y: CHART_RADIUS }
const RING_LABEL_OFFSET = 6
const RING_LABEL_PERPENDICULAR_OFFSET = 4
const CATEGORY_LABEL_RADIUS_OFFSET = 12
const LINGER_DURATION = 1000

// ============================================================================
// COMPUTED VALUES
// ============================================================================

const validatedStepCount = computed(() => Math.max(2, props.stepCount))

const categoriesWithDefaultValues = ref([...props.categories])

// Watch for changes in props.categories to update local state
watch(
  () => props.categories,
  (newCategories) => {
    categoriesWithDefaultValues.value = [...newCategories]
  },
  { deep: true }
)

// ============================================================================
// CHART GEOMETRY COMPUTATIONS
// ============================================================================

const stepCircles = computed(() => {
  return Array.from({ length: validatedStepCount.value + 1 }, (_, index) => ({
    x: CHART_CENTER.x,
    y: CHART_CENTER.y,
    radius: (CHART_RADIUS / validatedStepCount.value) * index,
  }))
})

const selectedPoints = computed(() => {
  const activeLayer = props.layers.find(
    (layer) => layer.name === props.activeLayer
  )
  if (!activeLayer) return []

  return categoriesWithDefaultValues.value.map((category, index) => {
    const angle = calculateSpineAngle(index)
    const value =
      activeLayer.categoryValues[category.id] ??
      Math.ceil(validatedStepCount.value / 2)
    const radiusMultiplier = value / validatedStepCount.value
    return {
      x: CHART_CENTER.x + CHART_RADIUS * radiusMultiplier * Math.cos(angle),
      y: CHART_CENTER.y + CHART_RADIUS * radiusMultiplier * Math.sin(angle),
    }
  })
})

const allLayerPolygons = computed(() => {
  return props.layers
    .map((layer) => ({
      layerName: layer.name,
      layerColor: layer.color,
      points: categoriesWithDefaultValues.value.map((category, index) => {
        const angle = calculateSpineAngle(index)
        const value =
          layer.categoryValues[category.id] ??
          Math.ceil(validatedStepCount.value / 2)
        const radiusMultiplier = value / validatedStepCount.value
        return {
          x: CHART_CENTER.x + CHART_RADIUS * radiusMultiplier * Math.cos(angle),
          y: CHART_CENTER.y + CHART_RADIUS * radiusMultiplier * Math.sin(angle),
        }
      }),
    }))
    .sort((a, b) => {
      // Active layer should be last (rendered on top)
      if (a.layerName === props.activeLayer) return 1
      if (b.layerName === props.activeLayer) return -1
      return 0
    })
})

const selectedPointsString = computed(() => {
  return selectedPoints.value.map((point) => `${point.x},${point.y}`).join(' ')
})

const spines = computed(() => {
  return categoriesWithDefaultValues.value.map((_, index) => {
    const angle = calculateSpineAngle(index)
    const outerRadius = CHART_RADIUS
    const innerRadius = CHART_RADIUS / validatedStepCount.value
    return {
      outer: {
        x: CHART_CENTER.x + outerRadius * Math.cos(angle),
        y: CHART_CENTER.y + outerRadius * Math.sin(angle),
      },
      inner: {
        x: CHART_CENTER.x + innerRadius * Math.cos(angle),
        y: CHART_CENTER.y + innerRadius * Math.sin(angle),
      },
    }
  })
})

const allValidPointsByCategory = computed(() => {
  return categoriesWithDefaultValues.value.map((_, categoryIndex) => {
    const angle = calculateSpineAngle(categoryIndex)
    return Array.from({ length: validatedStepCount.value }, (_, ringIndex) => {
      const ringRadius =
        (CHART_RADIUS / validatedStepCount.value) * (ringIndex + 1)
      return {
        x: CHART_CENTER.x + ringRadius * Math.cos(angle),
        y: CHART_CENTER.y + ringRadius * Math.sin(angle),
        ringIndex,
        categoryIndex,
      }
    })
  })
})

const categoryLabels = computed(() => {
  return categoriesWithDefaultValues.value.map((category, index) => {
    const angle = calculateSpineAngle(index)
    const labelRadius = CHART_RADIUS + CATEGORY_LABEL_RADIUS_OFFSET
    const angleDegrees = (angle * 180) / Math.PI + 90

    return {
      x: CHART_CENTER.x + labelRadius * Math.cos(angle),
      y: CHART_CENTER.y + labelRadius * Math.sin(angle),
      category: category.name,
      angle: angleDegrees,
      textAnchor: 'middle',
    }
  })
})

const ringLabels = computed(() => {
  const activeSpine = getActiveSpineIndex()

  return categoriesWithDefaultValues.value.flatMap((_, categoryIndex) => {
    const angle = calculateSpineAngle(categoryIndex)

    return Array.from({ length: validatedStepCount.value }, (_, ringIndex) => {
      const ringRadius =
        (CHART_RADIUS / validatedStepCount.value) * (ringIndex + 1)
      const circleX = CHART_CENTER.x + ringRadius * Math.cos(angle)
      const circleY = CHART_CENTER.y + ringRadius * Math.sin(angle)

      const perpendicularAngle = angle + Math.PI / 2
      const labelX =
        circleX -
        RING_LABEL_OFFSET * Math.cos(angle) -
        RING_LABEL_PERPENDICULAR_OFFSET * Math.cos(perpendicularAngle)
      const labelY =
        circleY -
        RING_LABEL_OFFSET * Math.sin(angle) -
        RING_LABEL_PERPENDICULAR_OFFSET * Math.sin(perpendicularAngle)

      return {
        x: labelX,
        y: labelY,
        value: ringIndex + 1,
        angle: (angle * 180) / Math.PI + 90,
        categoryIndex,
        ringIndex,
        visible: categoryIndex === activeSpine,
      }
    })
  })
})

// ============================================================================
// DRAG INTERACTION STATE
// ============================================================================

const isDragging = ref(false)
const draggedSpineIndex = ref(-1)
const hoveredSpineIndex = ref(-1)
const lingeringSpineIndex = ref(-1)
const svgRef = ref<SVGElement>()
let lingerTimer: ReturnType<typeof setTimeout> | null = null

const draggedPointValue = computed(() => {
  const activeLayer = props.layers.find(
    (layer) => layer.name === props.activeLayer
  )
  if (!activeLayer) return 1

  const category = categoriesWithDefaultValues.value[draggedSpineIndex.value]
  if (!category) return 1

  return (
    activeLayer.categoryValues[category.id] ??
    Math.ceil(validatedStepCount.value / 2)
  )
})

const draggedPoint = computed(() => {
  const spineIndex = draggedSpineIndex.value
  const ringIndex = (draggedPointValue.value ?? 1) - 1

  if (allValidPointsByCategory.value[spineIndex]?.[ringIndex]) {
    return allValidPointsByCategory.value[spineIndex][ringIndex]
  }
  return { x: 0, y: 0 }
})

// ============================================================================
// UTILITY FUNCTIONS
// ============================================================================

const calculateSpineAngle = (index: number): number => {
  return (
    ((2 * Math.PI) / categoriesWithDefaultValues.value.length) * index +
    (props.alignTo === 'polarZero' ? 0 : -Math.PI / 2)
  )
}

const getActiveSpineIndex = (): number => {
  if (isDragging.value) return draggedSpineIndex.value
  if (hoveredSpineIndex.value !== -1) return hoveredSpineIndex.value
  return lingeringSpineIndex.value
}

const convertMouseToSVGCoordinates = (event: MouseEvent) => {
  if (!svgRef.value) return { x: 0, y: 0 }

  const rect = svgRef.value.getBoundingClientRect()
  const svgEl = svgRef.value as SVGSVGElement
  const viewBox = svgEl.viewBox.baseVal
  const scaleX = viewBox.width / rect.width
  const scaleY = viewBox.height / rect.height

  return {
    x: (event.clientX - rect.left) * scaleX + viewBox.x,
    y: (event.clientY - rect.top) * scaleY + viewBox.y,
  }
}

const findClosestValidPoint = (
  svgCoords: { x: number; y: number },
  validPoints: any[]
) => {
  return validPoints.reduce(
    (closest, point) => {
      const distance = Math.sqrt(
        (svgCoords.x - point.x) ** 2 + (svgCoords.y - point.y) ** 2
      )
      return distance < closest.distance ? { point, distance } : closest
    },
    { point: validPoints[0], distance: Infinity }
  )
}

const clearLingerTimer = () => {
  if (lingerTimer) {
    clearTimeout(lingerTimer)
    lingerTimer = null
  }
}

// ============================================================================
// EVENT HANDLERS
// ============================================================================

const startDraggingPoint = (event: MouseEvent, spineIndex: number) => {
  clearLingerTimer()
  draggedSpineIndex.value = spineIndex
  isDragging.value = true
  event.preventDefault()
}

const updateDraggedPointPosition = (event: MouseEvent) => {
  if (!isDragging.value) return

  const spineIndex = draggedSpineIndex.value
  const svgCoords = convertMouseToSVGCoordinates(event)
  const validPoints = allValidPointsByCategory.value[spineIndex]

  if (!validPoints) return

  const closestPoint = findClosestValidPoint(svgCoords, validPoints)

  if (
    closestPoint.point &&
    categoriesWithDefaultValues.value[spineIndex] !== undefined
  ) {
    const newValue = closestPoint.point.ringIndex + 1
    const activeLayer = props.layers.find(
      (layer) => layer.name === props.activeLayer
    )
    if (activeLayer) {
      activeLayer.categoryValues[
        categoriesWithDefaultValues.value[spineIndex].id
      ] = newValue
    }
  }
}

const stopDraggingPoint = () => {
  isDragging.value = false
  draggedSpineIndex.value = -1
}

const showRingLabelsOnHover = (spineIndex: number) => {
  clearLingerTimer()
  hoveredSpineIndex.value = spineIndex
  lingeringSpineIndex.value = spineIndex
}

const hideRingLabelsOnLeave = () => {
  hoveredSpineIndex.value = -1
  lingerTimer = setTimeout(() => {
    lingeringSpineIndex.value = -1
    lingerTimer = null
  }, LINGER_DURATION)
}

const selectPointByClick = (categoryIndex: number, pointIndex: number) => {
  if (categoriesWithDefaultValues.value[categoryIndex] === undefined) return

  const activeLayer = props.layers.find(
    (layer) => layer.name === props.activeLayer
  )
  if (activeLayer) {
    activeLayer.categoryValues[
      categoriesWithDefaultValues.value[categoryIndex].id
    ] = pointIndex + 1
  }
}
</script>

<template>
  <div :class="['radar-chart', { dragging: isDragging }]">
    <ClientOnly>
      <svg
        ref="svgRef"
        @mousemove="updateDraggedPointPosition"
        @mouseup="stopDraggingPoint"
        width="600"
        height="600"
        :viewBox="`-20 -20 ${CHART_RADIUS * 2 + 40} ${CHART_RADIUS * 2 + 40}`"
      >
        <!-- Step Circles -->
        <circle
          v-for="(stepCircle, idx) in stepCircles"
          :key="idx"
          :cx="stepCircle.x"
          :cy="stepCircle.y"
          :r="stepCircle.radius"
          fill="transparent"
          :stroke="props.radarSkeleton.strokeColor"
          :stroke-width="props.radarSkeleton.strokeWidth"
          :opacity="props.radarSkeleton.strokeOpacity"
        />

        <!-- Spines -->
        <line
          v-for="(spine, idx) in spines"
          :key="idx"
          :x1="spine.outer.x"
          :y1="spine.outer.y"
          :x2="spine.inner.x"
          :y2="spine.inner.y"
          :stroke="props.radarSkeleton.strokeColor"
          :stroke-width="props.radarSkeleton.strokeWidth"
          :opacity="props.radarSkeleton.strokeOpacity"
        />

        <!-- Valid Points -->
        <template
          v-for="(category, idx) in allValidPointsByCategory"
          :key="idx"
        >
          <circle
            v-for="(point, pointIndex) in category"
            @click="selectPointByClick(idx, pointIndex)"
            @mouseenter="showRingLabelsOnHover(idx)"
            @mouseleave="hideRingLabelsOnLeave"
            :key="`${idx}-${point.ringIndex}`"
            :cx="point.x"
            :cy="point.y"
            :r="2"
            :stroke="props.radarSkeleton.strokeColor"
            :stroke-width="props.radarSkeleton.strokeWidth"
            class="valid-point"
            fill="white"
          />
        </template>

        <!-- Layer Polygons -->
        <polygon
          v-for="layer in allLayerPolygons"
          :key="layer.layerName"
          :points="layer.points.map((p) => `${p.x},${p.y}`).join(' ')"
          :stroke="layer.layerColor"
          :stroke-width="layer.layerName === props.activeLayer ? '2.5' : '1.5'"
          :stroke-opacity="layer.layerName === props.activeLayer ? '1' : '0.66'"
          :fill="
            layer.layerName === props.activeLayer
              ? layer.layerColor
              : 'transparent'
          "
          :fill-opacity="layer.layerName === props.activeLayer ? '0.10' : '0'"
          pointer-events="none"
        />

        <!-- Selected Points -->
        <circle
          v-for="(point, idx) in selectedPoints"
          @mousedown="startDraggingPoint($event, idx)"
          @mouseenter="showRingLabelsOnHover(idx)"
          @mouseleave="hideRingLabelsOnLeave"
          :key="idx"
          :cx="point.x"
          :cy="point.y"
          :r="3"
          class="selected-point"
          fill="white"
          :stroke="
            props.layers.find((l) => l.name === props.activeLayer)?.color ||
            '#3b82f6'
          "
          stroke-width="1.5"
        />

        <!-- Ring Labels -->
        <text
          v-for="ringLabel in ringLabels"
          :key="`ring-label-${ringLabel.categoryIndex}-${ringLabel.ringIndex}`"
          :x="ringLabel.x"
          :y="ringLabel.y"
          text-anchor="middle"
          dominant-baseline="middle"
          :class="['ring-label', { 'ring-label-visible': ringLabel.visible }]"
          font-size="6"
          fill="#666"
          :transform="`rotate(${ringLabel.angle} ${ringLabel.x} ${ringLabel.y})`"
        >
          {{ ringLabel.value }}
        </text>

        <!-- Category Labels -->
        <text
          v-for="(label, idx) in categoryLabels"
          :key="`label-${idx}`"
          :x="label.x"
          :y="label.y"
          :text-anchor="label.textAnchor"
          dominant-baseline="middle"
          class="category-label"
          font-size="8"
          fill="#999"
          :transform="`rotate(${label.angle} ${label.x} ${label.y})`"
        >
          {{ label.category }}
        </text>
      </svg>
    </ClientOnly>
  </div>
</template>

<style scoped>
.radar-chart {
  display: inline-block;
  padding: 16px;
}

.dragging:hover {
  cursor: grabbing;
}

.selected-point {
  cursor: grab;
}

.selected-point:active {
  cursor: grabbing;
}

.valid-point {
  cursor: pointer;
}

.valid-point:hover {
  stroke: v-bind(
    'props.layers.find(l => l.name === props.activeLayer)?.color || "#3b82f6"'
  );
  stroke-width: 1;
}

.ring-label {
  font-family:
    -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  font-weight: 400;
  pointer-events: none;
  user-select: none;
  opacity: 0;
  transition:
    opacity 0.2s ease-in-out,
    transform 0.2s ease-in-out;
}

.ring-label-visible {
  opacity: 0.6;
}

.category-label {
  font-family:
    -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  font-weight: 400;
  pointer-events: none;
  user-select: none;
  opacity: 0.7;
}
</style>
