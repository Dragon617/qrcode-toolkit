<script setup lang="ts">
import { debounce } from 'perfect-debounce'
import { sendQRCodeToCompare } from '~/logic/utils'
import type { Segment, State } from '~/logic/types'
import { HightlightFactor, compareSegments, segmentImage } from '~/logic/diff'
import { defaultCompareState, showDownloadDialog, showGridHelper } from '~/logic/state'
import { view } from '~/logic/view'

const props = defineProps<{
  state: State
}>()

const uploadTarget = ref<'image' | 'qrcode'>()
const fullState = computed(() => props.state)
const state = computed(() => props.state.compare)
const dataurl = computed({
  get() { return props.state.uploaded.image },
  set(v) { props.state.uploaded.image = v },
})
const dataUrlQRCode = computed({
  get() { return props.state.uploaded.qrcode },
  set(v) { props.state.uploaded.qrcode = v },
})

const gridSize = computed(() => state.value.gridSize)
const gridCellSize = computed(() => 100 / gridSize.value)
const gridMarginSize = computed(() => state.value.gridMarginSize)

const highlightMismatch = ref(false)
const highlightMismatchBorder = ref(false)
const selectedSegment = ref<Segment | null>(null)

const imageSegments = shallowRef<Segment[]>()
const qrcodeSegments = shallowRef<Segment[]>()

const debouncedImageSeg = debounce(async () => {
  imageSegments.value = dataurl.value
    ? await segmentImage(
      dataurl.value,
      gridSize.value,
    )
    : undefined
}, 800)

const debouncedQRCodeSeg = debounce(async () => {
  qrcodeSegments.value = dataUrlQRCode.value
    ? await segmentImage(
      dataUrlQRCode.value,
      gridSize.value,
    )
    : undefined
}, 800)

watch(
  () => [dataurl.value, state.value.gridSize],
  (newValue, oldValue) => {
    if (newValue.every((v, i) => v === oldValue?.[i]))
      return
    debouncedImageSeg()
  },
  { immediate: true },
)

watch(
  () => [dataUrlQRCode.value, state.value.gridSize],
  (newValue, oldValue) => {
    if (newValue.every((v, i) => v === oldValue?.[i]))
      return
    debouncedQRCodeSeg()
  },
  { immediate: true },
)

const { isOverDropZone } = useDropZone(document.body, {
  onDrop(files) {
    if (view.value !== 'compare')
      return
    if (!files || !uploadTarget.value)
      return

    const file = files[0]
    if (file.type === 'image/png' || file.type === 'image/jpeg') {
      const reader = new FileReader()
      reader.onload = () => {
        const data = reader.result as string
        if (uploadTarget.value === 'image') {
          fullState.value.uploaded.image = data
        }
        else if (uploadTarget.value === 'qrcode') {
          fullState.value.uploaded.qrcode = data
          showGridHelper.value = true
        }
      }
      reader.readAsDataURL(file)
    }
  },
  onLeave() {
    uploadTarget.value = undefined
  },
  onOver(_, event) {
    if (view.value !== 'compare')
      return
    if (!isOverDropZone.value)
      return

    const chain = Array.from(document.elementsFromPoint(event.clientX, event.clientY))
    if (chain.find(el => el.id === 'upload-zone-image'))
      uploadTarget.value = 'image'
    else if (chain.find(el => el.id === 'upload-zone-qrcode'))
      uploadTarget.value = 'qrcode'
    else
      uploadTarget.value = undefined
  },
})

const diff = computed(() => {
  if (!imageSegments.value || !qrcodeSegments.value)
    return null
  return compareSegments(imageSegments.value, qrcodeSegments.value, state.value)
})

function onSegmentHover(segment: Segment) {
  selectedSegment.value = segment
}

function onSegmentLeave(segment: Segment) {
  setTimeout(() => {
    if (selectedSegment.value === segment)
      selectedSegment.value = null
  }, 100)
}

function reset() {
  // eslint-disable-next-line no-alert
  if (confirm('Are you sure to reset all state?'))
    Object.assign(state.value, defaultCompareState())
}

const filter = computed(() => {
  const items = [
    state.value.grayscale && 'saturate(0)',
    `contrast(${state.value.contrast}%)`,
    `brightness(${state.value.brightness}%)`,
    state.value.blur && `blur(${state.value.blur}px)`,
  ]
  return items.filter(Boolean).join(' ')
})

function toggleHighContrast() {
  if (state.value.grayscale) {
    state.value.grayscale = false
    state.value.contrast = 100
  }
  else {
    state.value.grayscale = true
    state.value.contrast = 500
  }
}
</script>

<template>
  <div grid="~ cols-[35rem_1fr] gap-4" relative>
    <div flex="~ col gap-4" relative>
      <div v-if="dataurl" border="~ base rounded-lg" relative aspect-ratio-1 of-hidden>
        <img
          :src="dataurl"
          absolute inset-0 h-full w-full object-cover
          :style="{ filter }"
        >
        <div
          v-if="state.pixelView && imageSegments"
          relative h-full w-full object-cover
          :style="{ filter }"
        >
          <div
            v-for="s of imageSegments"
            :key="s.index"
            :style="{
              position: 'absolute',
              left: `${s.x * gridCellSize}%`,
              top: `${s.y * gridCellSize}%`,
              width: `${gridCellSize}%`,
              height: `${gridCellSize}%`,
              background: s.hex,
            }"
          />
        </div>

        <img
          v-if="dataUrlQRCode && state.overlay"
          :src="dataUrlQRCode"
          absolute inset-0 h-full w-full object-cover
          :style="{
            filter: `blur(${state.blur}px)`,
            opacity: state.overlayOpacity,
            mixBlendMode: state.overlayBlendMode as any,
          }"
        >

        <GridLines
          v-if="state.grid"
          v-bind="{
            gridSize,
            gridColor: state.gridColor,
            gridOpacity: state.gridOpacity,
            gridMarginSize,
            gridMarginColor: state.gridColor,
          }"
        />

        <!-- Mismatches -->
        <template v-if="highlightMismatch && diff">
          <div
            v-for="s of diff.mismatchDark"
            :key="s.index"
            :style="{
              position: 'absolute',
              zIndex: 200,
              left: `${s.x * gridCellSize}%`,
              top: `${s.y * gridCellSize}%`,
              width: `${gridCellSize}%`,
              height: `${gridCellSize}%`,
              background: `rgba(255,255,255,${Math.abs(diff.lightLuminance - s.luminance) * HightlightFactor * state.correctionOpacity / 255})`,
              border: highlightMismatchBorder ? '1px solid #00FFFF' : 'none',
              filter: highlightMismatchBorder ? 'none' : `blur(${state.correctionBlur}px)`,
              mixBlendMode: highlightMismatchBorder ? 'none' : state.correctionBlendMode as any,
            }"
          />
          <div
            v-for="s of diff.mismatchLight"
            :key="s.index"
            :style="{
              position: 'absolute',
              zIndex: 200,
              left: `${s.x * gridCellSize}%`,
              top: `${s.y * gridCellSize}%`,
              width: `${gridCellSize}%`,
              height: `${gridCellSize}%`,
              background: `rgba(0,0,0,${Math.abs(diff.darkLuminance - s.luminance) * HightlightFactor * state.correctionOpacity / 255})`,
              border: highlightMismatchBorder ? '1px solid #FFFF00' : 'none',
              filter: highlightMismatchBorder ? 'none' : `blur(${state.correctionBlur}px)`,
              mixBlendMode: highlightMismatchBorder ? 'none' : state.correctionBlendMode as any,
            }"
          />
        </template>

        <div
          v-if="selectedSegment"
          :style="{
            position: 'absolute',
            zIndex: 200,
            left: `${selectedSegment.x * gridCellSize}%`,
            top: `${selectedSegment.y * gridCellSize}%`,
            width: `${gridCellSize}%`,
            height: `${gridCellSize}%`,
            border: '2px solid #FFFF00',
          }"
        >
          <div
            flex="~ gap-1 items-center" absolute bg-black:60 px1 py0.5
            border="~ base rounded"
            text-xs shadow backdrop-blur-2
            :style="{
              top: 'calc(100% + 5px)',
              left: '50%',
              transform: 'translateX(-50%)',
            }"
          >
            <div
              h-3 w-3 rounded border="~ base"
              :style="{
                background: selectedSegment.hex,
              }"
            />
            {{ (selectedSegment.luminance / 255 * 100).toFixed(1) }}%
          </div>
        </div>
      </div>

      <div v-if="dataurl" border="~ base rounded" flex="~ col gap-2" p4>
        <OptionItem title="灰度">
          <OptionCheckbox v-model="state.grayscale" />
        </OptionItem>
        <OptionItem title="对比度" @reset="state.contrast = 100">
          <OptionSlider v-model="state.contrast" :min="0" :max="1000" :step="10" unit="%" :default="100" />
        </OptionItem>
        <OptionItem title="亮度" @reset="state.brightness = 100">
          <OptionSlider v-model="state.brightness" :min="0" :max="1000" :step="10" unit="%" :default="100" />
        </OptionItem>
        <OptionItem title="模糊">
          <OptionSlider v-model="state.blur" :min="0" :max="10" :step="1" unit="px" />
        </OptionItem>
        <OptionItem title="像素化">
          <OptionCheckbox v-model="state.pixelView" />
        </OptionItem>

        <div border="t base" my1 />

        <OptionItem title="网格" description="Toggle Grid View">
          <OptionCheckbox v-model="state.grid" />
          <div flex-auto />
          <button
            v-if="dataUrlQRCode"
            text-xs text-button
            @click="showGridHelper = true"
          >
            <div i-ri-artboard-2-line />
            对齐网格
          </button>
        </OptionItem>
        <OptionItem title="网格尺寸" nested>
          <OptionSlider v-model="state.gridSize" :min="10" :max="100" :step="1" />
        </OptionItem>
        <SettingsMargin v-model="state.gridMarginSize" />
        <template v-if="state.grid">
          <OptionItem title="不透明度" nested>
            <OptionSlider v-model="state.gridOpacity" :min="0" :max="1" :step="0.01" />
          </OptionItem>
          <OptionItem title="颜色" nested>
            <OptionColor v-model="state.gridColor" />
          </OptionItem>
        </template>

        <template v-if="dataUrlQRCode">
          <div border="t base" my1 />

          <OptionItem title="覆盖图">
            <OptionCheckbox v-model="state.overlay" />
          </OptionItem>
          <template v-if="state.overlay">
            <OptionItem title="不透明度" nested>
              <OptionSlider v-model="state.overlayOpacity" :min="0" :max="1" :step="0.01" />
            </OptionItem>
            <OptionItem title="混合模式" nested>
              <OptionSelectGroup
                v-model="state.overlayBlendMode"
                :options="['normal', 'overlay', 'darken', 'lighten', 'difference']"
              />
            </OptionItem>
          </template>
        </template>
      </div>

      <div v-if="dataurl">
        <button
          text-sm op75 text-button hover:text-red hover:op100
          @click="reset()"
        >
          Reset State
        </button>
      </div>

      <div grid="~ cols-[1fr_max-content_1fr] gap-4" border="~ base rounded" px2 py8>
        <div flex="~ col items-center gap-2">
          <div text-sm op75>
            目标图像
          </div>
          <ImageDrop v-model="dataurl" title="Target Image" />
        </div>
        <div border="l base" />
        <div flex="~ col gap-2 items-center">
          <div text-sm op75>
            来源二维码
          </div>
          <ImageDrop
            v-model="dataUrlQRCode"
            title="Source QRCode"
            @update:model-value="e => showGridHelper = e ? true : false"
          />
          <button
            text-sm op75 text-button
            @click="sendQRCodeToCompare(fullState)"
          >
            <div i-ri-qr-code-line />
            从生成器中应用
          </button>
        </div>
      </div>
    </div>

    <!-- Right Panel -->
    <div v-if="dataurl" border="~ base rounded" flex="~ col gap-2" h-max p4>
      <template v-if="!dataUrlQRCode">
        <div text-sm flex="~ items-center gap-2">
          <button relative text-button>
            <div i-ri-upload-line />
            Upload QR Code
            <ImageUpload v-model="dataUrlQRCode" />
          </button>
          <div>
            to start comparing
          </div>
        </div>
      </template>
      <template v-else-if="!diff">
        <div flex="~ items-center gap-2" animate-pulse text-center>
          <div>
            Loading...
          </div>
        </div>
      </template>
      <template v-else>
        <div relative pb4>
          <div
            h-10px of-hidden rounded
            :style="{
              background: 'linear-gradient(to right, #000000, #FFFFFF)',
            }"
          />
          <div
            i-ri-arrow-up-s-fill absolute top-1px text-xl
            :style="{
              left: `${diff.avarageLuminance / 255 * 100}%`,
              transform: 'translateX(-50%)',
            }"
          />
          <div
            i-ri-arrow-up-s-fill absolute top-1px text-xl
            :style="{
              left: `${diff.lightLuminance / 255 * 100}%`,
              transform: 'translateX(-50%)',
            }"
          />
          <div
            i-ri-arrow-up-s-fill absolute top-1px text-xl
            :style="{
              left: `${diff.darkLuminance / 255 * 100}%`,
              transform: 'translateX(-50%)',
            }"
          />
          <div
            v-if="selectedSegment"
            i-ri-arrow-up-s-fill absolute top-1px text-xl text-yellow
            :style="{
              left: `${selectedSegment.luminance / 255 * 100}%`,
              transform: 'translateX(-50%)',
            }"
          />
        </div>
        <div grid="~ cols-[250px_110px]">
          <div op50>
            平均亮度
          </div><div>{{ (diff.avarageLuminance / 255 * 100).toFixed(1) }}%</div>
          <div op50>
            白色的平均亮度
          </div><div>{{ (diff.lightLuminance / 255 * 100).toFixed(1) }}%</div>
          <div op50>
            黑暗的平均亮度
          </div><div>{{ (diff.darkLuminance / 255 * 100).toFixed(1) }}%</div>
          <div op50>
            不匹配节点
          </div><div>{{ diff.mismatchDark.length + diff.mismatchLight.length }} / {{ diff.mainSegments.length }}</div>
        </div>

        <template v-if="diff.mismatchDark.length || diff.mismatchLight.length">
          <div my2 h-1px w-20 border-t border-base />

          <div flex="~ gap-2 wrap">
            <button
              text-sm text-button
              @pointerenter="highlightMismatch = true; highlightMismatchBorder = false"
              @pointerleave="highlightMismatch = false"
            >
              <div i-ri-bring-to-front />
              预览更正
            </button>
            <button
              text-sm text-button
              @pointerenter="highlightMismatch = true; highlightMismatchBorder = true"
              @pointerleave="highlightMismatch = false"
            >
              <div i-ri-search-2-line />
              突出显示不匹配
            </button>
            <button
              text-sm text-button
              @click="showDownloadDialog = true"
            >
              <div i-ri-download-line />
              下载
            </button>
          </div>
        </template>

        <div
          flex="~ col gap-2" px-1 py2
          @pointerenter="highlightMismatch = true; highlightMismatchBorder = false"
          @pointerleave="highlightMismatch = false"
        >
          <SettingsCorrection :state="state" />
        </div>

        <template v-if="diff.mismatchLight.length">
          <div my2 h-1px w-20 border-t border-base />
          <div op75>
            Dark Mismatch ({{ diff.mismatchLight.length }})
          </div>
          <div flex="~ wrap gap-1">
            <div v-for="s in diff.mismatchLight" :key="s.index">
              <SegmentViewer
                :segment="s"
                @pointerenter="onSegmentHover(s)"
                @pointerleave="onSegmentLeave(s)"
              />
            </div>
          </div>
        </template>
        <template v-if="diff.mismatchDark.length">
          <div my2 h-1px w-20 border-t border-base />
          <div op75>
            Light Mismatch ({{ diff.mismatchDark.length }})
          </div>
          <div flex="~ wrap gap-1">
            <div v-for="s in diff.mismatchDark" :key="s.index">
              <SegmentViewer
                :segment="s"
                @pointerenter="onSegmentHover(s)"
                @pointerleave="onSegmentLeave(s)"
              />
            </div>
          </div>
        </template>
      </template>
      <!-- <div>{{ scanResultQRCode }}</div> -->
      <!-- <div>{{ scanResultImage }}</div> -->
    </div>

    <!-- Action Panel -->
    <div
      v-if="dataurl" flex="~ col gap-2" absolute
      :style="{ right: 'calc(100% + 1rem)' }"
    >
      <VTooltip placement="left" distance="10">
        <button icon-button title="Toggle grid" @click="state.grid = !state.grid">
          <div i-ri-artboard-2-line :class="state.grid ? '' : 'op30'" />
        </button>
        <template #popper>
          <div text-sm>
            Toggle Grid View
          </div>
        </template>
      </VTooltip>
      <VTooltip placement="left" distance="10">
        <button icon-button title="Toggle overlay" @click="state.overlay = !state.overlay">
          <div i-ri-qr-code-fill :class="state.overlay ? '' : 'op30'" />
        </button>
        <template #popper>
          <div text-sm>
            Toggle QR Code Overlay
          </div>
        </template>
      </VTooltip>
      <VTooltip placement="left" distance="10">
        <button icon-button title="Toggle high contrast" @click="toggleHighContrast()">
          <div i-ri-contrast-line :class="state.grayscale ? '' : 'op30'" />
        </button>
        <template #popper>
          <div text-sm>
            Toggle High Contrast
          </div>
        </template>
      </VTooltip>
      <VTooltip v-if="imageSegments" placement="left" distance="10">
        <button icon-button title="Toggle high contrast" @click="state.pixelView = !state.pixelView">
          <div i-ri-grid-line :class="state.pixelView ? '' : 'op30'" />
        </button>
        <template #popper>
          <div text-sm>
            Toggle Pixel View
          </div>
        </template>
      </VTooltip>

      <div my4 h-1px border="t base" />

      <VTooltip placement="left" distance="10">
        <div relative icon-button>
          <div i-ri-upload-2-line op50 />
          <ImageUpload v-model="dataurl" />
        </div>
        <template #popper>
          <div text-sm>
            Upload Image
          </div>
        </template>
      </VTooltip>
    </div>
  </div>

  <DialogDownload
    v-if="showDownloadDialog"
    v-model="showDownloadDialog"
    :state="fullState"
    :diff="diff"
  />

  <div v-if="isOverDropZone" grid="~ cols-2" fixed bottom-0 left-0 right-0 top-0 z-200 bg-black:20 backdrop-blur-10>
    <div
      id="upload-zone-image" flex="~ col gap-3 items-center justify-center" m10 mr-1 op40
      :class="uploadTarget === 'image' ? 'bg-gray:20 op100 border-base' : ''"
      border="3 dashed transparent rounded-xl"
    >
      <div i-carbon-image text-20 />
      <div text-xl>
        Upload as target image
      </div>
    </div>
    <div
      id="upload-zone-qrcode" flex="~ col gap-3 items-center justify-center" m10 ml-1 op40
      :class="uploadTarget === 'qrcode' ? 'bg-gray:20 op100 border-base' : ''"
      border="3 dashed transparent rounded-xl"
    >
      <div i-carbon-qr-code text-20 />
      <div text-xl>
        Upload as QR Code reference
      </div>
    </div>
  </div>
</template>
