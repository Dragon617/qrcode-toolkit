<script setup lang="ts">
import type { State } from '~/logic/types'

const props = defineProps<{
  modelValue: boolean
  state: State
}>()

const emit = defineEmits<{
  (event: 'update:modelValue', value: boolean): void
}>()

const gridSize = ref(props.state.compare.gridSize)
const gridMarginSize = ref(props.state.compare.gridMarginSize)

function close() {
  emit('update:modelValue', false)
}

function cancel() {
  close()
}

function apply() {
  props.state.compare.gridSize = gridSize.value
  props.state.compare.gridMarginSize = gridMarginSize.value
  cancel()
}
</script>

<template>
  <dialog
    :open="modelValue"
    class="fixed inset-0 z-50000"
    @close="cancel()"
  >
    <div
      class="fixed inset-0 bg-black/20 backdrop-blur-10"
      @click="apply()"
    />
    <div class="relative z-10 max-w-full w-[35rem] rounded-lg p-4 bg-base" flex="~ col gap-4" border="~ base">
      <h1 text-xl>
        Align Grid
      </h1>
      <div border="~ base rounded-lg" relative aspect-ratio-1 of-hidden>
        <img
          :src="state.uploaded.qrcode"
          absolute inset-0 h-full w-full object-cover op65
        >

        <GridLines
          v-bind="{
            gridSize,
            gridColor: 'red',
            gridOpacity: 0.8,
            gridMarginSize,
            gridMarginColor: 'lime',
            darkenMargin: true,
          }"
        />
      </div>

      <p>
        滑动网格滑块，使红色网格线与所有像素对齐。
      </p>
      <p>
        滑动边缘滑块，使绿色框与二维码主体对齐。
      </p>

      <div flex="~ col gap-2">
        <OptionItem title="网格">
          <OptionSlider v-model="gridSize" :min="0" :max="50" :step="1" />
        </OptionItem>

        <SettingsMargin v-model="gridMarginSize" />
      </div>

      <div flex="~ gap-2 justify-end" mt-5>
        <button px5 op75 text-button @click="cancel()">
          取消
        </button>
        <button w-25 px5 text-button @click="apply()">
          应用
        </button>
      </div>
    </div>
  </dialog>
</template>
