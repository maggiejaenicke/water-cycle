<template>
  <div
    ref="root"
    class="wc-diagram-layers"
    @pointerover="handlePointerOver"
    @pointerout="handlePointerOut"
    @focusin="handleFocusIn"
    @focusout="handleFocusOut"
  >
    <!-- Layer 1: background artwork (non-interactive raster, language-neutral) -->
    <img
      class="wc-layer wc-layer--background"
      :src="backgroundSrc"
      alt="Illustrated diagram of the water cycle showing the major pools and fluxes of water on Earth."
      draggable="false"
    >

    <!-- Layer 2: arrows (non-interactive, decorative, language-neutral) -->
    <img
      class="wc-layer wc-layer--arrows"
      :src="arrowsSrc"
      alt=""
      aria-hidden="true"
      draggable="false"
    >

    <!-- Layer 3: interactive labels. vite-svg-loader renders this inline as
         real DOM so we can attach hover/focus handlers to [data-term].
         :key forces a re-render (and re-application of a11y attributes) when
         the language changes, once a language-specific SVG is provided. -->
    <component
      :is="activeLabelsSvg"
      :key="language"
      class="wc-layer wc-layer--labels"
    />
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import LabelsSvgEn from '@/assets/images/WCD_all_labels.svg?component'
// When the Spanish labels SVG is available, import it here and wire it into
// `activeLabelsSvg` below:
//   import LabelsSvgEs from '@/assets/images/WCD_all_labels_es.svg?component'
import backgroundSrc from '@/assets/images/WCD_image_background.png'
import arrowsSrc from '@/assets/images/WCD_all_arrows.svg?url'
import terms from '@/assets/text/terms.json'

const props = defineProps({
  // Current app language ('en' | 'es'). Drives which definitions are shown and
  // (eventually) which labels SVG is rendered.
  language: {
    type: String,
    default: 'en'
  },
  // Audience level ('beginner' | 'advanced'). Current scope: advanced.
  level: {
    type: String,
    default: 'advanced'
  }
})

// Emits the active term (with screen-space anchor) so the parent can render a
// tooltip outside the zoom/pan transform (keeping tooltip text unscaled).
const emit = defineEmits(['term-enter', 'term-leave'])

const root = ref(null)

// Select the labels SVG for the active language. Until a Spanish SVG exists we
// render the English artwork for both languages (definitions still localize via
// the `language` prop). To enable Spanish labels, import LabelsSvgEs above and
// return it for 'es' here.
const activeLabelsSvg = computed(() => {
  switch (props.language) {
    // case 'es': return LabelsSvgEs
    default: return LabelsSvgEn
  }
})

function titleFor(termId) {
  const entry = terms[termId]
  if (!entry) return termId
  return [entry.label, entry.qualifier].filter(Boolean).join(' ')
}

function definitionFor(termId) {
  const entry = terms[termId]
  return entry?.definitions?.[props.language]?.[props.level] ?? ''
}

function termIdFromTarget(target) {
  if (!(target instanceof Element)) return null
  const labelEl = target.closest('[data-term]')
  return labelEl?.getAttribute('data-term') ?? null
}

// Compute a screen-space anchor (viewport coords) for the label element.
// Reports both edges so the parent can place the tooltip above or below.
function anchorFor(target) {
  const labelEl = target instanceof Element ? target.closest('[data-term]') : null
  if (!labelEl) return null
  const rect = labelEl.getBoundingClientRect()
  return {
    x: rect.left + rect.width / 2,
    top: rect.top,
    bottom: rect.bottom
  }
}

function emitEnter(termId, target) {
  const definition = definitionFor(termId)
  if (!definition) return
  emit('term-enter', {
    id: termId,
    title: titleFor(termId),
    definition,
    anchor: anchorFor(target)
  })
}

function handlePointerOver(event) {
  const termId = termIdFromTarget(event.target)
  if (termId) emitEnter(termId, event.target)
}

function handlePointerOut(event) {
  const fromTerm = termIdFromTarget(event.target)
  const toTerm = termIdFromTarget(event.relatedTarget)
  if (fromTerm && fromTerm !== toTerm) emit('term-leave')
}

function handleFocusIn(event) {
  const termId = termIdFromTarget(event.target)
  if (termId) emitEnter(termId, event.target)
}

function handleFocusOut() {
  emit('term-leave')
}

// Make tagged labels keyboard-focusable and screen-reader friendly without
// re-editing the generated SVG. Re-applied whenever the inline SVG changes
// (initial mount and on language switch, via the :key re-render).
function applyLabelAccessibility() {
  const labels = root.value?.querySelectorAll('[data-term]') ?? []
  labels.forEach((el) => {
    const termId = el.getAttribute('data-term')
    el.setAttribute('tabindex', '0')
    el.setAttribute('role', 'button')
    el.setAttribute('aria-label', `${titleFor(termId)}. ${definitionFor(termId)}`)
    el.style.cursor = 'pointer'
  })
}

onMounted(applyLabelAccessibility)

// Re-apply after the SVG re-renders for a new language, and clear any open
// tooltip so stale text does not linger across the switch.
watch(() => props.language, async () => {
  emit('term-leave')
  await nextTick()
  applyLabelAccessibility()
})
</script>

<style scoped lang="scss">
// The background image is in normal flow and drives the component's intrinsic
// size (mirrors the original diagram <img> rules). The container shrink-wraps
// that rendered size so the absolutely-positioned arrows/labels overlay it
// exactly (all three share the same artboard).
.wc-diagram-layers {
  position: relative;
  display: inline-block;
  max-width: 100%;
  max-height: 100%;
  line-height: 0; // remove inline-block baseline gap under the image
}

.wc-layer--background {
  display: block;
  width: auto;
  height: auto;
  max-width: 100%;
  max-height: 100%;
  pointer-events: none;
  user-select: none;
}

.wc-layer--arrows,
.wc-layer--labels {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  display: block;
}

.wc-layer--arrows {
  object-fit: contain;
  pointer-events: none;
  user-select: none;
  z-index: 2;
}

.wc-layer--labels {
  z-index: 3;
}
.wc-layer--background {
  z-index: 1;
}

.wc-layer--labels :deep([data-term]) {
  transition: opacity 0.1s ease-out;
  // The exported label text uses fill:none + stroke (a white halo), so the
  // default SVG hit-testing (visiblePainted) only responds on the thin strokes,
  // leaving dead zones inside letters and between words. Force the whole label
  // area to be hoverable.
  pointer-events: all;
}

.wc-layer--labels :deep([data-term] text),
.wc-layer--labels :deep([data-term] tspan) {
  pointer-events: all;
}

.wc-layer--labels :deep([data-term]:hover),
.wc-layer--labels :deep([data-term]:focus-visible) {
  opacity: 0.7;
  outline: none;
}
</style>
