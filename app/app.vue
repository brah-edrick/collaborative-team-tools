<script setup>
import RadarChart from './components/RadarChart.vue'

const categories = ref([
  {
    name: 'Collaboration',
    id: '1',
    description: 'Description 1',
  },
  {
    name: 'Execution',
    id: '2',
    description: 'Description 2',
  },
  {
    name: 'Autonomy',
    id: '3',
    description: 'Description 3',
  },
  {
    name: 'Quality',
    id: '4',
    description: 'Description 4',
  },
  {
    name: 'Security',
    id: '5',
    description: 'Description 5',
  },
  {
    name: 'Belonging',
    id: '6',
    description: 'Description 6',
  },
  {
    name: 'Pride & Craft',
    id: '7',
    description: 'Description 7',
  },
])

const layers = ref([
  {
    name: 'layer1',
    color: '#3b82f6',
    editable: true,
    categoryValues: {
      1: 4,
      2: 5,
      3: 1,
      4: 3,
      5: 3,
      6: 2,
      7: 2,
    },
  },
  {
    name: 'layer2',
    color: '#ef4444',
    editable: true,
    categoryValues: {
      1: 2,
      2: 3,
      3: 4,
      4: 5,
      5: 2,
      6: 4,
      7: 3,
    },
  },
  {
    name: 'layer3',
    color: '#10b981',
    editable: false,
    categoryValues: {
      1: 5,
      2: 4,
      3: 2,
      4: 3,
      5: 4,
      6: 5,
      7: 1,
    },
  },
])

const activeLayer = ref('layer1')
</script>

<template>
  <div>
    <div style="margin-bottom: 20px">
      <label>Active Layer:</label>
      <select v-model="activeLayer">
        <option value="layer1">Layer 1</option>
        <option value="layer2">Layer 2</option>
        <option value="layer3">Layer 3</option>
      </select>
    </div>

    <RadarChart
      :categories="categories"
      :layers="layers"
      :active-layer="activeLayer"
      align-to="top"
      :step-count="7"
    />
  </div>
</template>
