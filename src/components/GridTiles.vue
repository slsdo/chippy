<script setup lang="ts">
import { computed, ref, onMounted, onUnmounted, watch, onBeforeUnmount } from 'vue'
import type { Hex } from '../lib/hex'
import type { Layout } from '../lib/layout'
import { useDragDrop } from '../composables/useDragDrop'
import { useGridStore } from '../stores/grid'
import { useCharacterStore } from '../stores/character'
import { useMapEditorStore } from '../stores/mapEditor'
import { State } from '../lib/types/state'
import { getHexFillColor } from '../utils/stateFormatting'
import { Team } from '../lib/types/team'
import { useGridEvents } from '../composables/useGridEvents'

interface Props {
  hexes: Hex[]
  layout: Layout
  width?: number | string
  height?: number | string
  rotation?: number
  scaleX?: number
  scaleY?: number
  skewX?: number
  skewY?: number
  centerX?: number
  centerY?: number
  strokeWidth?: number
  showHexIds?: boolean
  showCoordinates?: boolean
  hexIdFontSize?: number
  coordinateFontSize?: number
  textColor?: string
  coordinateColor?: string
  textRotation?: number
  hexFillColor?: string
  hexStrokeColor?: string
  isMapEditorMode?: boolean
  selectedMapEditorState?: State
}

const props = withDefaults(defineProps<Props>(), {
  width: 500,
  height: 500,
  rotation: 0,
  scaleX: 1,
  scaleY: 1,
  skewX: 0,
  skewY: 0,
  centerX: 250,
  centerY: 250,
  strokeWidth: 2,
  showHexIds: true,
  showCoordinates: true,
  hexIdFontSize: 18,
  coordinateFontSize: 8,
  textColor: '#222',
  coordinateColor: '#555',
  textRotation: 0,
  hexFillColor: '#fff',
  hexStrokeColor: '#ccc',
  isMapEditorMode: false,
  selectedMapEditorState: State.DEFAULT,
})

const gridEvents = useGridEvents()

const {
  handleDragOver,
  handleDrop,
  hasCharacterData,
  draggedCharacter,
  hoveredHexId,
  isDragging,
  setHoveredHex,
  setDropHandled,
} = useDragDrop()
const gridStore = useGridStore()
const characterStore = useCharacterStore()
const mapEditorStore = useMapEditorStore()

// Track which hex is currently being hovered (non-drag)
const hoveredHex = ref<number | null>(null)

// Map editor drag-to-paint state
// Enables continuous painting while dragging mouse across hexes
const isMapEditorDragging = ref(false)
const paintedHexes = ref(new Set<number>()) // Tracks painted hexes to avoid duplicates
let lastPaintTime = 0
const PAINT_THROTTLE_MS = 50 // Performance: throttle painting to every 50ms

// Watch for changes in position-based hex detection
watch(hoveredHexId, (newHexId) => {
  if (isDragging.value && newHexId !== null) {
    // Trigger drop validation for the detected hex
    const hex = gridStore.getHexById(newHexId)
    // We could emit drag over events here if needed
  }
})

const gridTransform = computed(() => {
  const transforms: string[] = []
  if (props.rotation !== 0) {
    transforms.push(`rotate(${props.rotation},${props.centerX},${props.centerY})`)
  }
  if (props.skewX !== 0) {
    transforms.push(`skewX(${props.skewX})`)
  }
  if (props.skewY !== 0) {
    transforms.push(`skewY(${props.skewY})`)
  }
  if (props.scaleX !== 1 || props.scaleY !== 1) {
    transforms.push(`scale(${props.scaleX},${props.scaleY})`)
  }
  return transforms.join(' ')
})

const textTransform = (hex: Hex) => {
  if (props.textRotation === 0) return ''
  const pos = props.layout.hexToPixel(hex)
  return `rotate(${props.textRotation},${pos.x},${pos.y})`
}

const getHexFill = (hex: Hex) => {
  const state = gridStore.grid.getState(hex)

  return getHexFillColor(state) || props.hexFillColor
}

const shouldShowHexId = (hex: Hex) => {
  const state = gridStore.grid.getState(hex)
  return state !== State.BLOCKED
}

// Mouse hover handling functions
const handleHexMouseEnter = (hex: Hex) => {
  hoveredHex.value = hex.getId()

  // Map editor drag-to-paint with throttling
  if (props.isMapEditorMode && isMapEditorDragging.value) {
    const hexId = hex.getId()
    const now = Date.now()

    if (!paintedHexes.value.has(hexId) && now - lastPaintTime >= PAINT_THROTTLE_MS) {
      const success = mapEditorStore.setHexState(hexId, props.selectedMapEditorState)
      if (success) {
        paintedHexes.value.add(hexId)
        lastPaintTime = now
      }
    }
  }
}

const handleHexMouseLeave = (hex: Hex) => {
  if (hoveredHex.value === hex.getId()) {
    hoveredHex.value = null
  }
}

// Map editor mouse handlers for drag-to-paint functionality
// Enables painting multiple hexes by holding mouse button and dragging
const handleMapEditorMouseDown = () => {
  if (props.isMapEditorMode) {
    isMapEditorDragging.value = true
    paintedHexes.value.clear() // Reset painted hexes for new drag session
  }
}

const handleMapEditorMouseUp = () => {
  if (props.isMapEditorMode) {
    isMapEditorDragging.value = false
    paintedHexes.value.clear() // Cleanup after drag session
  }
}

// Clean up map editor state on unmount
onBeforeUnmount(() => {
  isMapEditorDragging.value = false
  paintedHexes.value.clear()
})

/**
 * Hybrid drag detection: combines SVG events with position-based detection
 * to handle drops when character portraits block tile events.
 */
const handleHexDragOver = (event: DragEvent, hex: Hex) => {
  if (hasCharacterData(event)) {
    handleDragOver(event)
    // Sync with global hover state for visual feedback
    setHoveredHex(hex.getId())
  }
}

const handleHexDragLeave = (event: DragEvent, hex: Hex) => {
  // Only clear if position detection confirms we left this hex
  const currentDetectedHex = hoveredHexId.value
  if (currentDetectedHex !== hex.getId()) {
    setHoveredHex(null)
  }
}

const handleHexDrop = (event: DragEvent, hex: Hex) => {
  // Prevent event from bubbling up to global handlers
  event.stopPropagation()
  event.preventDefault()

  const dropResult = handleDrop(event)

  // Hover state is managed by position-based detection

  if (dropResult) {
    const { character, characterName } = dropResult

    setDropHandled(true) // Prevent duplicate processing

    // Grid-to-grid character moves have sourceHexId from overlay drag handlers
    if (character.sourceHexId !== undefined) {
      const sourceHexId = character.sourceHexId
      const targetHexId = hex.getId()

      // Swap if target is occupied, otherwise move
      if (characterStore.isHexOccupied(targetHexId)) {
        characterStore.swapCharacters(sourceHexId, targetHexId)
      } else {
        // Empty target - regular move
        characterStore.moveCharacter(sourceHexId, targetHexId, characterName)
      }
    } else {
      // Character selection placement
      const hexId = hex.getId()
      const tile = gridStore.getTile(hexId)
      const state = tile.state

      // Auto-assign team based on tile state
      let team: Team
      if (state === State.AVAILABLE_ALLY || state === State.OCCUPIED_ALLY) {
        team = Team.ALLY
      } else if (state === State.AVAILABLE_ENEMY || state === State.OCCUPIED_ENEMY) {
        team = Team.ENEMY
      } else {
        return
      }

      // Validate team capacity
      if (!characterStore.canPlaceCharacter(characterName, team)) {
        return
      }

      const success = characterStore.placeCharacterOnHex(hexId, characterName, team)
      if (!success) {
        return
      }
    }
  }
}

const getHexDropClass = (hex: Hex) => {
  const hexId = hex.getId()
  const isOccupied = characterStore.isHexOccupied(hexId)
  // Use position-based hover detection instead of SVG event-based detection
  const isDragHover = isDragging.value && hoveredHexId.value === hexId

  // Validate drop zone for visual feedback
  let validDropZone = false
  if (isDragHover && draggedCharacter.value) {
    const tile = gridStore.getTile(hexId)
    const state = tile.state

    // Check if tile accepts characters
    if (
      state === State.AVAILABLE_ALLY ||
      state === State.OCCUPIED_ALLY ||
      state === State.AVAILABLE_ENEMY ||
      state === State.OCCUPIED_ENEMY
    ) {
      // Get tile team for validation
      const tileTeam =
        state === State.AVAILABLE_ALLY || state === State.OCCUPIED_ALLY ? Team.ALLY : Team.ENEMY

      // Grid moves: always allow (just repositioning existing characters)
      if (draggedCharacter.value.sourceHexId !== undefined) {
        // This is a character being moved from another hex on the grid
        validDropZone = true
      } else {
        // Character selection: check team capacity
        validDropZone = characterStore.canPlaceCharacter(draggedCharacter.value.name, tileTeam)
      }
    }
  }

  return {
    'drop-target': true,
    occupied: isOccupied,
    'drag-hover': isDragHover,
    'invalid-drop': isDragHover && !validDropZone,
    hover: hoveredHex.value === hexId,
  }
}

const isElevated = (hex: Hex) => {
  return characterStore.isHexOccupied(hex.getId())
}

const regularHexes = computed(() => props.hexes.filter((hex) => !isElevated(hex)))
const elevatedHexes = computed(() => props.hexes.filter((hex) => isElevated(hex)))

// Hover state is now managed by position-based detection
const handleDragEnded = () => {
  // No longer needed - position-based system handles cleanup
}

onMounted(() => {
  document.addEventListener('drag-ended', handleDragEnded)
})

onUnmounted(() => {
  document.removeEventListener('drag-ended', handleDragEnded)
})
</script>

<template>
  <svg
    :width="width"
    :height="height"
    class="grid-tiles"
    :class="{ 'map-editor-mode': isMapEditorMode }"
    @mousedown="handleMapEditorMouseDown"
    @mouseup="handleMapEditorMouseUp"
    @mouseleave="handleMapEditorMouseUp"
  >
    <defs>
      <slot name="defs" />
    </defs>
    <g :transform="gridTransform">
      <!-- Regular hexes (render first, behind elevated hexes) -->
      <g v-for="hex in regularHexes" :key="hex.getId()" class="grid-tile">
        <polygon
          :points="
            layout
              .polygonCorners(hex)
              .map((p) => `${p.x},${p.y}`)
              .join(' ')
          "
          :fill="getHexFill(hex)"
          :stroke="hexStrokeColor"
          :stroke-width="strokeWidth"
        />
        <text
          v-if="showHexIds && shouldShowHexId(hex)"
          :x="layout.hexToPixel(hex).x"
          :y="layout.hexToPixel(hex).y + 6"
          text-anchor="middle"
          :font-size="hexIdFontSize"
          :fill="textColor"
          font-family="monospace"
          :transform="textTransform(hex)"
        >
          {{ hex.getId() }}
        </text>
        <text
          v-if="showCoordinates"
          :x="layout.hexToPixel(hex).x"
          :y="layout.hexToPixel(hex).y + 18"
          text-anchor="middle"
          :font-size="coordinateFontSize"
          :fill="coordinateColor"
          font-family="monospace"
          :transform="textTransform(hex)"
        >
          ({{ hex.q }},{{ hex.r }},{{ hex.s }})
        </text>
      </g>

      <!-- Elevated hexes (render above regular hexes, but below character components) -->
      <g v-for="hex in elevatedHexes" :key="`elevated-${hex.getId()}`" class="grid-tile">
        <polygon
          :points="
            layout
              .polygonCorners(hex)
              .map((p) => `${p.x},${p.y}`)
              .join(' ')
          "
          :fill="getHexFill(hex)"
          :stroke="hexStrokeColor"
          :stroke-width="strokeWidth"
        />
        <text
          v-if="showHexIds && shouldShowHexId(hex)"
          :x="layout.hexToPixel(hex).x"
          :y="layout.hexToPixel(hex).y + 6"
          text-anchor="middle"
          :font-size="hexIdFontSize"
          :fill="textColor"
          font-family="monospace"
          :transform="textTransform(hex)"
        >
          {{ hex.getId() }}
        </text>
        <text
          v-if="showCoordinates"
          :x="layout.hexToPixel(hex).x"
          :y="layout.hexToPixel(hex).y + 18"
          text-anchor="middle"
          :font-size="coordinateFontSize"
          :fill="coordinateColor"
          font-family="monospace"
          :transform="textTransform(hex)"
        >
          ({{ hex.q }},{{ hex.r }},{{ hex.s }})
        </text>
      </g>

      <!-- Character components and other content -->
      <slot />

      <!-- 
        Invisible event layer - MUST be rendered last to be topmost layer
        This ensures drag and drop events are captured even when hovering over characters
        All character visual elements have pointer-events: none to allow events to pass through
      -->
      <g
        v-for="hex in hexes"
        :key="`event-${hex.getId()}`"
        class="grid-event-layer"
        :class="getHexDropClass(hex)"
      >
        <polygon
          :points="
            layout
              .polygonCorners(hex)
              .map((p) => `${p.x},${p.y}`)
              .join(' ')
          "
          fill="transparent"
          stroke="transparent"
          stroke-width="0"
          @click="gridEvents.emit('hex:click', hex)"
          @mouseenter="handleHexMouseEnter(hex)"
          @mouseleave="handleHexMouseLeave(hex)"
          @dragover="handleHexDragOver($event, hex)"
          @dragleave="handleHexDragLeave($event, hex)"
          @drop="handleHexDrop($event, hex)"
        />
      </g>
    </g>
  </svg>
</template>

<style scoped>
.grid-tiles {
  max-width: 100%;
  height: auto;
}

.grid-tiles.map-editor-mode {
  cursor: crosshair;
}

.grid-tile {
  cursor: pointer;
}

.grid-event-layer {
  cursor: pointer;
  pointer-events: all;
  /* Ensure event layer can receive drop events even with HTML overlays above */
}

/* Ensure event layer polygons can receive all pointer events including drops */
.grid-event-layer polygon {
  pointer-events: all;
  transition:
    fill 0.2s ease,
    stroke 0.2s ease,
    stroke-width 0.2s ease;
}

/* Occupied tiles - red border on event layer */
.grid-event-layer.drop-target.occupied polygon {
  stroke: #999;
  stroke-width: 3;
}

/* Drag hover states - highest priority with !important */
.grid-event-layer.drop-target.drag-hover:not(.occupied):not(.invalid-drop) polygon {
  fill: rgba(232, 245, 232, 0.3) !important;
  stroke: #36958e !important;
  stroke-width: 3 !important;
  filter: drop-shadow(0 0 8px rgba(76, 175, 80, 0.4));
}

.grid-event-layer.drop-target.drag-hover.occupied:not(.invalid-drop) polygon {
  fill: rgba(255, 232, 232, 0.3) !important;
  stroke: #ff9800 !important;
  stroke-width: 3 !important;
  filter: drop-shadow(0 0 8px rgba(255, 152, 0, 0.4));
}

/* Invalid drop zone styling */
.grid-event-layer.drop-target.drag-hover.invalid-drop polygon {
  fill: rgba(255, 193, 193, 0.3) !important;
  stroke: #c05b4d !important;
  stroke-width: 3 !important;
  filter: drop-shadow(0 0 8px rgba(244, 67, 54, 0.4));
}

/* Regular hover (when not dragging) */
.grid-event-layer.drop-target:not(.occupied):not(.drag-hover).hover polygon {
  fill: rgba(240, 248, 240, 0.3);
  stroke: #36958e;
  stroke-width: 3;
}

/* Visual grid tiles remain clean without interaction styling */
.grid-tile.hover polygon {
  fill-opacity: 0.8;
}
</style>
