<template>
  <open-seadragon
    @mousewheel.prevent="osdWheel"
    :options="prepped_osd_options"
    @viewer-bound="callback"
    :pixelated="true"
  />
</template>

<script>
import OpenSeadragon from './OpenSeadragon.vue'
import OSD from 'openseadragon'

export default {
  components: { OpenSeadragon },
  props: {
    osd_store: Object,
    editor_mode: Boolean, // editor_mode is scroll to pan, ctrl+scroll to zoom (like figma)
    osd_options: Object,
  },
  emits: ['viewer-bound'],
  data() {
    const { editor_mode } = this
    const prepped_osd_options = {
      maxZoomPixelRatio: editor_mode ? 8 : 4,
      navigatorAutoFade: false,
      showNavigator: true,
      showZoomControl: false,
      showHomeControl: false,
      showFullPageControl: false,
      showRotationControl: false,
      debugmode: false,
      clickTimeThreshold: 1000,
      gestureSettingsMouse: {
        clickToZoom: false,
        dblClickToZoom: false,
      },
      ...this.osd_options,
    }
    if (prepped_osd_options.mouseNavEnabled === undefined) {
      prepped_osd_options.mouseNavEnabled = !editor_mode
    }
    return { prepped_osd_options, viewer: null }
  },
  watch: {
    osd_options: {
      deep: true,
      handler() {
        this.viewer.setMouseNavEnabled(!!this.osd_options.mouseNavEnabled)
      },
    },
    editor_mode() {
      const { viewer } = this
      const enabled = this.osd_options?.mouseNavEnabled ?? true
      viewer.setMouseNavEnabled(!this.editor_mode && enabled)
      viewer.viewport.maxZoomPixelRatio = this.editor_mode ? 8 : 4
    },
  },
  methods: {
    osdWheel(event) {
      if (!this.editor_mode || !this.osd_options.mouseNavEnabled) {
        return
      }
      // Loosely adapted from Openseadragon.Viewer's onCanvasDragEnd onCanvasScroll
      const { viewer } = this
      const viewport = viewer.viewport
      if (event.ctrlKey) {
        const box = viewer.container.getBoundingClientRect()
        const position = new OSD.Point(event.pageX - box.left, event.pageY - box.top)
        const factor = Math.pow(viewer.zoomPerScroll, -event.deltaY / 20)
        viewport.zoomBy(factor, viewport.pointFromPixel(position, true))
      } else {
        const center = viewport.pixelFromPoint(viewport.getCenter(true))
        const target = viewport.pointFromPixel(
          new OSD.Point(center.x + 3 * event.deltaX, center.y + 3 * event.deltaY),
        )
        viewport.panTo(target, false)
      }
      viewport.applyConstraints()
    },
    callback(viewer) {
      this.viewer = viewer
      this.osd_store?.bindViewer(viewer)
      this.$emit('viewer-bound', viewer)
    },
  },
}
</script>
