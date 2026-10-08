<template>
  <div class="wrapper">
    <!-- eslint-disable vue/attribute-hyphenation -->
    <ExcalidrawComponent
      :theme="theme"
      :initialData="initialData"
      :onChange="onChange"
      :UIOptions="uiOptions"
    >
      <!-- eslint-enable vue/attribute-hyphenation -->
      <WelcomeScreen />
    </ExcalidrawComponent>
  </div>
</template>

<script setup lang="ts">
import { Excalidraw, THEME, serializeAsJSON } from '@excalidraw/excalidraw'
import '@excalidraw/excalidraw/index.css'
import { applyPureReactInVue } from 'veaury'
import { type AppConfigObject, useThemeStore } from '@ownclouders/web-pkg'
import { storeToRefs } from 'pinia'
import { computed, unref } from 'vue'
import WelcomeScreen from '../components/WelcomeScreen.vue'
import { type Resource } from '@ownclouders/web-client'

const props = defineProps<{
  resource: Resource
  currentContent: string
  applicationConfig: AppConfigObject
}>()
const emit = defineEmits<{ 'update:currentContent': [string] }>()

const assetsPath = props.applicationConfig.assetsPath

/**
 * The subset of the `.excalidraw` file format we care about.
 *
 * The format is defined by Excalidraw itself and produced by `serializeAsJSON`,
 * which this extension bundles. It is an object, not an array:
 *
 *   { type: "excalidraw", version: 2, source, elements, appState, files }
 *
 * See https://docs.excalidraw.com/docs/codebase/json-schema
 */
type ExcalidrawScene = {
  elements?: unknown[]
  appState?: Record<string, unknown>
  files?: Record<string, unknown>
}

/**
 * Read the file content into a scene.
 *
 * Two shapes are accepted:
 *
 *  - An object, which is the official format written by every Excalidraw app
 *    (excalidraw.com, self-hosted Excalidraw, and `serializeAsJSON` below).
 *  - A bare array of elements, which is what this extension wrote up to and
 *    including v0.1.0. Files saved by that version keep working.
 *
 * A parse failure must not propagate. Excalidraw sets `isLoading` to true
 * before it reads `initialData` and only clears it once the scene is restored,
 * so throwing here leaves the editor on its loading spinner forever with no
 * indication of what went wrong.
 */
const parseScene = (content: string): ExcalidrawScene => {
  if (content === '') {
    return { elements: [] }
  }

  let parsed: unknown
  try {
    parsed = JSON.parse(content)
  } catch (error) {
    console.error('Could not parse the Excalidraw file', error)
    return { elements: [] }
  }

  if (Array.isArray(parsed)) {
    return { elements: parsed }
  }

  if (parsed !== null && typeof parsed === 'object') {
    return parsed as ExcalidrawScene
  }

  console.error(
    'Unexpected Excalidraw file content, expected an object or an array'
  )
  return { elements: [] }
}

const scene = parseScene(props.currentContent)

/**
 * `theme` is owned by ownCloud Web, which passes it down as a prop from the
 * theme store. Dropping it from the restored state keeps a board that was saved
 * in dark mode from forcing dark mode on a light instance.
 */
const initialAppState = (() => {
  if (!scene.appState) {
    return undefined
  }
  const { theme: _theme, ...rest } = scene.appState
  return rest
})()

const initialData = {
  elements: Array.isArray(scene.elements) ? scene.elements : [],
  files: scene.files ?? {},
  appState: initialAppState,
}

const uiOptions = Object.freeze({
  canvasActions: {
    loadScene: false,
    export: false,
    saveToActiveFile: false,
    toggleTheme: false,
    saveAsImage: false,
    changeViewBackgroundColor: false,
  },
})

const ExcalidrawComponent = applyPureReactInVue(Excalidraw)

const themeStore = useThemeStore()
const { currentTheme } = storeToRefs(themeStore)

const theme = computed(() =>
  unref(currentTheme).isDark ? THEME.DARK : THEME.LIGHT
)

/**
 * Write the file content.
 *
 * `serializeAsJSON` is Excalidraw's own serializer, so the result is readable
 * by every other Excalidraw app. Serializing only `elements` would drop
 * `appState` and, more importantly, `files` — the embedded images — on every
 * save.
 *
 * `serializeAsJSON` runs the files through `filterOutDeletedFiles`, so images
 * belonging to removed elements are not carried along.
 */
const onChange = (
  elements: Parameters<typeof serializeAsJSON>[0],
  appState: Parameters<typeof serializeAsJSON>[1],
  files: Parameters<typeof serializeAsJSON>[2]
) => {
  if (elements.length === 0) {
    emit('update:currentContent', '')
    return
  }

  emit(
    'update:currentContent',
    serializeAsJSON(elements, appState, files, 'local')
  )
}

if (typeof assetsPath === 'string' && assetsPath !== '') {
  window.EXCALIDRAW_ASSET_PATH = assetsPath
}
</script>

<style scoped>
.wrapper {
  height: 100%;
  width: 100%;
}
</style>
