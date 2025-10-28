<script setup lang="ts">
import { ref, computed, watch } from 'vue'

interface Category {
  name: string
  id: string
  description: string
  value?: number
}

interface PolarSkeleton {
  strokeWidth: number
  strokeColor: string
  strokeOpacity: number
}

interface Props {
  categories: Category[]
  alignTo: 'top' | 'polarZero'
  stepCount?: number
  polarSkeleton?: PolarSkeleton
}

const props = withDefaults(defineProps<Props>(), {
  stepCount: 5,
  polarSkeleton: () => ({
    strokeWidth: 1,
    strokeColor: '#ccc',
    strokeOpacity: 1,
  }),
})

// Enforce minimum stepCount of 2
const validatedStepCount = computed(() => Math.max(2, props.stepCount))

// Create a copy of categories with default values
const categoriesWithDefaultValues = ref(
  props.categories.map((category) => ({
    ...category,
    value:
      category.value ?? Math.ceil(props.stepCount ? props.stepCount / 2 : 3),
  }))
)

// Watch for changes in props.categories to update local state
watch(
  () => props.categories,
  (newCategories) => {
    const defaultValue = Math.ceil(validatedStepCount.value / 2)
    categoriesWithDefaultValues.value = newCategories.map((category) => ({
      ...category,
      value: category.value ?? defaultValue,
    }))
  },
  { deep: true }
)

const radius = 100
const circleRadius = radius
const center = { x: radius, y: radius }

const stepCircleProps = computed(() => {
  return Array.from({ length: validatedStepCount.value + 1 }, (_, index) => {
    const x = center.x
    const y = center.y
    // Add offset to prevent zero ring at center
    const radius = (circleRadius / validatedStepCount.value) * index
    return { x, y, radius }
  })
})

const selectedPoints = computed(() => {
  return categoriesWithDefaultValues.value.map((category, index) => {
    const angle =
      ((2 * Math.PI) / categoriesWithDefaultValues.value.length) * index +
      (props.alignTo === 'polarZero' ? 0 : -Math.PI / 2)
    const x =
      center.x +
      radius * (category.value / validatedStepCount.value) * Math.cos(angle)
    const y =
      center.y +
      radius * (category.value / validatedStepCount.value) * Math.sin(angle)
    return { x, y }
  })
})

const selectedPointsString = computed(() => {
  return selectedPoints.value
    .map((point) => {
      return `${point.x},${point.y}`
    })
    .join(' ')
})

const spines = computed(() => {
  return categoriesWithDefaultValues.value.map((category, index) => {
    const angle =
      ((2 * Math.PI) / categoriesWithDefaultValues.value.length) * index +
      (props.alignTo === 'polarZero' ? 0 : -Math.PI / 2)
    const outerX = center.x + radius * Math.cos(angle)
    const innerX =
      center.x + ((radius * 1) / validatedStepCount.value) * Math.cos(angle)
    const outerY = center.y + radius * Math.sin(angle)
    const innerY =
      center.y + ((radius * 1) / validatedStepCount.value) * Math.sin(angle)
    return {
      outer: { x: outerX, y: outerY },
      inner: { x: innerX, y: innerY },
    }
  })
})

const allPossibleValidPointsByCategory = computed(() => {
  return Array.from({ length: categoriesWithDefaultValues.value.length }).map(
    (_, categoryIndex) => {
      const angle =
        ((2 * Math.PI) / categoriesWithDefaultValues.value.length) *
          categoryIndex +
        (props.alignTo === 'polarZero' ? 0 : -Math.PI / 2)
      return Array.from({ length: validatedStepCount.value }).map(
        (_, ringIndex) => {
          // Use the same radius calculation as stepCircleProps
          const ringRadius =
            (circleRadius / validatedStepCount.value) * (ringIndex + 1)
          const x = center.x + ringRadius * Math.cos(angle)
          const y = center.y + ringRadius * Math.sin(angle)
          return { x, y, ringIndex, categoryIndex }
        }
      )
    }
  )
})

const allPossibleValidPoints = computed(() => {
  return allPossibleValidPointsByCategory.value.flat()
})

/**
 * Draggable points logic and state management
 */

const draggedPointIndex = ref(-1) // which spoke are we dragging?
const isDragging = ref(false) // are we dragging a point?
const svgRef = ref<SVGElement>()

const handleMouseDown = (event: MouseEvent, index: number) => {
  draggedPointIndex.value = index
  isDragging.value = true
  event.preventDefault() // Prevent text selection
}

const getSVGCoordinates = (event: MouseEvent) => {
  if (!svgRef.value) return { x: 0, y: 0 }

  const rect = svgRef.value.getBoundingClientRect()
  // Use type assertion to access SVGSVGElement-specific properties
  const svgEl = svgRef.value as SVGSVGElement
  const scaleX = svgEl.viewBox.baseVal.width / rect.width
  const scaleY = svgEl.viewBox.baseVal.height / rect.height

  return {
    x: (event.clientX - rect.left) * scaleX,
    y: (event.clientY - rect.top) * scaleY,
  }
}

const handleMouseMove = (event: MouseEvent) => {
  if (isDragging.value) {
    const index = draggedPointIndex.value
    const svgCoords = getSVGCoordinates(event)

    if (!allPossibleValidPointsByCategory.value[index]) return
    const validPoints = allPossibleValidPointsByCategory.value[index]

    const closestPoint = validPoints.reduce(
      (closest, point) => {
        const distance = Math.sqrt(
          (svgCoords.x - point.x) ** 2 + (svgCoords.y - point.y) ** 2
        )
        return distance < closest.distance ? { point, distance } : closest
      },
      { point: validPoints[0], distance: Infinity }
    )

    // Update the category value based on the closest point's ring index
    if (
      closestPoint.point &&
      categoriesWithDefaultValues.value[index] !== undefined
    ) {
      const newValue = closestPoint.point.ringIndex + 1 // ringIndex is 0-based, value is 1-based
      categoriesWithDefaultValues.value[index].value = newValue
    }
  }
}

const handleMouseUp = () => {
  isDragging.value = false
  draggedPointIndex.value = -1
}
</script>

<template>
  <div class="polar-chart">
    <ClientOnly>
      <svg
        ref="svgRef"
        @mousemove="handleMouseMove"
        @mouseup="handleMouseUp"
        width="400"
        height="400"
        :viewBox="`-2 -2 ${radius * 2.1} ${radius * 2.1}`"
      >
        <circle
          v-for="(stepCircle, idx) in stepCircleProps"
          :key="idx"
          :cx="stepCircle.x"
          :cy="stepCircle.y"
          :r="stepCircle.radius"
          fill="transparent"
          :stroke="props.polarSkeleton.strokeColor"
          :stroke-width="props.polarSkeleton.strokeWidth"
          :opacity="props.polarSkeleton.strokeOpacity"
        />
        <line
          v-for="(spine, idx) in spines"
          :key="idx"
          :x1="spine.outer.x"
          :y1="spine.outer.y"
          :x2="spine.inner.x"
          :y2="spine.inner.y"
          :stroke="props.polarSkeleton.strokeColor"
          :stroke-width="props.polarSkeleton.strokeWidth"
          :opacity="props.polarSkeleton.strokeOpacity"
        />

        <circle
          v-for="(point, idx) in allPossibleValidPoints"
          :key="idx"
          :cx="point.x"
          :cy="point.y"
          :r="2"
          :stroke="props.polarSkeleton.strokeColor"
          :stroke-width="props.polarSkeleton.strokeWidth"
          fill="white"
        />
        <polygon
          :points="selectedPointsString"
          fill="none"
          stroke="blue"
          stroke-width="3"
          opacity="0.5"
        />
        <circle
          v-for="(point, idx) in selectedPoints"
          @mousedown="handleMouseDown($event, idx)"
          :key="idx"
          :cx="point.x"
          :cy="point.y"
          :r="4"
          class="draggable-point"
          fill="white"
          stroke="#333"
          stroke-width="2"
        />
      </svg>
    </ClientOnly>
  </div>
</template>
<style scoped>
.polar-chart {
  display: inline-block;
  padding: 16px;
  &:active {
    cursor: grabbing;
  }
}
.draggable-point {
  cursor: grab;
  &:active {
    cursor: grabbing;
  }
}
</style>
