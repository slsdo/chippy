<script setup lang="ts">
import Character from './Character.vue'
import type { CharacterType } from '../types/character'

defineProps<{
  character: CharacterType
  characterImage: string
  icons: { [key: string]: string }
}>()
</script>

<template>
  <div class="profile">
    <Character :character="character" :characterImage="characterImage" />
    <div class="info">
      <img :src="icons[`faction-${character.faction}`]" :alt="character.faction" class="icon" />
      <img :src="icons[`class-${character.class}`]" :alt="character.class" class="icon" />
    </div>
  </div>
</template>

<style scoped>
.profile {
  font-size: 1rem;
  font-weight: 600;
  text-align: center;
  margin-top: 0.25rem;
  color: #333;
}

.info {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
  padding-top: 0.4rem;
}

.info img {
  width: 24px;
  height: 24px;
  border: 1px solid #484848;
  border-radius: 50%;
}

@media (max-width: 480px) {
  .characters {
    gap: 0.5rem;
  }

  .profile img {
    width: 70px;
    height: 70px;
  }
}
</style>
