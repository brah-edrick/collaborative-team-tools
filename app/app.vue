<script setup>
import BasePolarChart from './components/BasePolarChart.vue'

const categories = ref([
  {
    name: 'Category 1',
    id: '1',
    description: 'Description 1',
  },
  {
    name: 'Category 2',
    id: '2',
    description: 'Description 2',
  },
  {
    name: 'Category 3',
    id: '3',
    description: 'Description 3',
  },
])

const pentagonCategories = ref([
  {
    name: 'Category 1',
    id: '1',
    description: 'Description 1',
  },
  {
    name: 'Category 2',
    id: '2',
    description: 'Description 2',
  },
  {
    name: 'Category 3',
    id: '3',
    description: 'Description 3',
  },
  {
    name: 'Category 4',
    id: '4',
    description: 'Description 4',
  },
  {
    name: 'Category 5',
    id: '5',
    description: 'Description 5',
  },
])

const heptagonWithValues = ref([
  {
    name: 'Category 1',
    id: '1',
    description: 'Description 1',
    value: 4,
  },
  {
    name: 'Category 2',
    id: '2',
    description: 'Description 2',
    value: 5,
  },
  {
    name: 'Category 3',
    id: '3',
    description: 'Description 3',
    value: 1,
  },
  {
    name: 'Category 4',
    id: '4',
    description: 'Description 4',
    value: 3,
  },
  {
    name: 'Category 5',
    id: '5',
    description: 'Description 5',
    value: 3,
  },
  {
    name: 'Category 6',
    id: '6',
    description: 'Description 6',
    value: 2,
  },
  {
    name: 'Category 7',
    id: '7',
    description: 'Description 7',
    value: 2,
  },
])
</script>

<template>
  <BasePolarChart
    :categories="categories"
    align-to="polarZero"
    :step-count="5"
  />
  <BasePolarChart
    :categories="categories"
    align-to="top"
    :step-count="5"
  />
  <BasePolarChart
    :categories="pentagonCategories"
    align-to="polarZero"
    :step-count="5"
  />
  <BasePolarChart
    :categories="pentagonCategories"
    align-to="top"
    :step-count="5"
  />
  <BasePolarChart
    :categories="heptagonWithValues"
    align-to="top"
    :step-count="7"
  />
</template>
