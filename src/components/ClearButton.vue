<script setup lang="ts">
const emit = defineEmits<{
  click: []
}>()

const handleClick = () => {
  emit('click')
}
</script>

<template>
  <button @click="handleClick" class="clear-all-btn">Clear All</button>
</template>

<style scoped>
.clear-all-btn {
  background: var(--color-danger);
  color: white;
  border: 2px solid var(--color-danger);
  padding: var(--spacing-md) var(--spacing-md);
  margin-right: 5px;
  border-radius: var(--radius-medium);
  cursor: pointer;
  font-size: 0.85rem;
  font-weight: 600;
  transition: all var(--transition-fast);
}

.clear-all-btn:hover {
  background: var(--color-danger-hover);
  border-color: var(--color-danger-hover);
}
</style>
