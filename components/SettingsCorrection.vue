<script setup lang="ts">
import type { ComparionState } from 'logic/types'

defineProps<{
  state: ComparionState
}>()
</script>

<template>
  <OptionItem title="模糊性阈值" @reset="state.diffThreshold = 3">
    <OptionSlider v-model="state.diffThreshold" :min="0.1" :max="20" :step="0.01" />
  </OptionItem>

  <OptionItem title="校正不透明度" @reset="state.correctionOpacity = 1">
    <OptionSlider v-model="state.correctionOpacity" :min="0" :max="4" :step="0.01" />
  </OptionItem>

  <OptionItem title="校正模糊" @reset="state.correctionBlur = 0">
    <OptionSlider v-model="state.correctionBlur" :min="0" :max="20" :step="0.05" />
  </OptionItem>

  <OptionItem title="校正混合">
    <OptionSelectGroup
      v-model="state.correctionBlendMode"
      :options="['none', 'overlay', 'darken', 'lighten', 'difference']"
    />
  </OptionItem>
</template>
