<template>
  <svg
    class="animate-spin"
    :class="[sizeClass, colorClass]"
    xmlns="http://www.w3.org/2000/svg"
    fill="none"
    viewBox="0 0 24 24"
  >
    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" />
    <path
      class="opacity-75"
      fill="currentColor"
      d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
    />
  </svg>
</template>

<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  size?: 'sm' | 'md' | 'lg' | 'xl'
  color?: 'white' | 'gray' | 'indigo' | 'red' | 'green'
}

const props = withDefaults(defineProps<Props>(), {
  size: 'md',
  color: 'white',
})

const sizeClass = computed(() => {
  const sizes = {
    sm: 'h-4 w-4',
    md: 'h-5 w-5',
    lg: 'h-6 w-6',
    xl: 'h-8 w-8',
  }
  return sizes[props.size]
})

const colorClass = computed(() => {
  const colors = {
    white: 'text-white',
    gray: 'text-gray-500',
    indigo: 'text-indigo-600',
    red: 'text-red-500',
    green: 'text-green-500',
  }
  return colors[props.color]
})
</script>
