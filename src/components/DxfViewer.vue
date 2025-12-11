<template>
<div class="canvasContainer" ref="canvasContainer" @mousemove="_OnMouseMove">
    <q-inner-loading :showing="isLoading" color="primary" style="z-index: 10"/>
    <div v-if="progress !== null" class="progress">
        <q-linear-progress color="primary" :indeterminate="progress < 0" :value="progress" />
        <div v-if="progressText !== null" class="progressText">{{progressText}}</div>
    </div>
    <div v-if="error !== null" class="error" :title="error">
        <q-icon name="warning" class="text-red" style="font-size: 4rem;" /> Error occurred: {{error}}
    </div>
    <div v-if="coordinates !== null" class="coordinates">
        <div class="coord-label">X: <span class="coord-value">{{ coordinates.x.toFixed(2) }}</span></div>
        <div class="coord-label">Y: <span class="coord-value">{{ coordinates.y.toFixed(2) }}</span></div>
        <div class="coord-label">Z: <span class="coord-value">{{ coordinates.z.toFixed(2) }}</span></div>
    </div>
</div>
</template>

<script>
import {DxfViewer} from "dxf-viewer"
import * as three from "three"
import DxfViewerWorker from "worker-loader!./DxfViewerWorker"

/** Events: all DxfViewer supported events (see DxfViewer.Subscribe()), prefixed with "dxf-". */
export default {
    name: "DxfViewer",

    props: {
        dxfUrl: {
            default: null
        },
        /** List of font URLs. Files should have TTF format. Fonts are used in the specified order,
         * each one is checked until necessary glyph is found. Text is not rendered if fonts are not
         * specified.
         */
        fonts: {
            default: null
        },
        options: {
            default() {
                return {
                    clearColor: new three.Color("#fff"),
                    autoResize: true,
                    colorCorrection: true,
                    sceneOptions: {
                        wireframeMesh: true
                    }
                }
            }
        }
    },

    data() {
        return {
            isLoading: false,
            progress: null,
            progressText: null,
            curProgressPhase: null,
            error: null,
            coordinates: null
        }
    },

    watch: {
        async dxfUrl(dxfUrl) {
            if (dxfUrl !== null) {
                await this.Load(dxfUrl)
            } else {
                this.dxfViewer.Clear()
                this.error = null
                this.isLoading = false
                this.progress = null
            }
        }
    },

    methods: {
        async Load(url) {
            this.isLoading = true
            this.error = null
            try {
                await this.dxfViewer.Load({
                    url,
                    fonts: this.fonts,
                    progressCbk: this._OnProgress.bind(this),
                    workerFactory: DxfViewerWorker
                })
                // Initialize coordinates to center after load
                if (this.dxfViewer.camera) {
                    const camera = this.dxfViewer.camera
                    this.coordinates = {
                        x: (camera.left + camera.right) / 2,
                        y: (camera.bottom + camera.top) / 2,
                        z: 0
                    }
                }
            } catch (error) {
                console.warn(error)
                this.error = error.toString()
            } finally {
                this.isLoading = false
                this.progressText = null
                this.progress = null
                this.curProgressPhase = null
            }
        },

        /** @return {DxfViewer} */
        GetViewer() {
            return this.dxfViewer
        },

        _OnMouseMove(event) {
            if (!this.dxfViewer) {
                return
            }
            
            const canvas = this.dxfViewer.GetCanvas()
            if (!canvas) {
                return
            }
            
            const rect = canvas.getBoundingClientRect()
            const x = event.clientX - rect.left
            const y = event.clientY - rect.top
            
            // Convert screen coordinates to normalized device coordinates (-1 to +1)
            const ndcX = (x / rect.width) * 2 - 1
            const ndcY = -(y / rect.height) * 2 + 1
            
            // Get camera from viewer
            const camera = this.dxfViewer.camera
            if (!camera) {
                return
            }
            
            // For orthographic camera, calculate world coordinates
            const camWidth = camera.right - camera.left
            const camHeight = camera.top - camera.bottom
            
            const worldX = camera.left + (ndcX + 1) * camWidth / 2
            const worldY = camera.bottom + (ndcY + 1) * camHeight / 2
            
            this.coordinates = {
                x: worldX,
                y: worldY,
                z: 0
            }
            
            // Emit event for parent components
            this.$emit('coordinates-update', this.coordinates)
        },

        _OnProgress(phase, size, totalSize) {
            if (phase !== this.curProgressPhase) {
                switch(phase) {
                case "font":
                    this.progressText = "Fetching fonts..."
                    break
                case "fetch":
                    this.progressText = "Fetching file..."
                    break
                case "parse":
                    this.progressText = "Parsing file..."
                    break
                case "prepare":
                    this.progressText = "Preparing rendering data..."
                    break
                }
                this.curProgressPhase = phase
            }
            if (totalSize === null) {
                this.progress = -1
            } else {
                this.progress = size / totalSize
            }
        }
    },

    mounted() {
        this.dxfViewer = new DxfViewer(this.$refs.canvasContainer, this.options)
        const Subscribe = eventName => {
            this.dxfViewer.Subscribe(eventName, e => this.$emit("dxf-" + eventName, e))
        }
        for (const eventName of ["loaded", "cleared", "destroyed", "resized", "pointerdown",
                                 "pointerup", "viewChanged", "message"]) {
            Subscribe(eventName)
        }
    },

    destroyed() {
        this.dxfViewer.Destroy()
        this.dxfViewer = null
    }
}
</script>

<style scoped lang="less">

.canvasContainer {
    position: relative;
    width: 100%;
    height: 100%;
    min-width: 100px;
    min-height: 100px;

    .progress {
        position: absolute;
        z-index: 20;
        width: 90%;
        margin: 20px 5%;

        .progressText {
            margin: 10px 20px;
            font-size: 14px;
            color: #262d33;
            text-align: center;
        }
    }

    .error {
        width: 100%;
        height: 100%;
        position: absolute;
        z-index: 20;
        padding: 30px;

        img {
            width: 24px;
            height: 24px;
            vertical-align: middle;
            margin: 4px;
        }
    }

    .coordinates {
        position: absolute;
        bottom: 10px;
        right: 10px;
        background: rgba(255, 255, 255, 0.95);
        border: 1px solid #ccc;
        border-radius: 4px;
        padding: 8px 12px;
        font-family: monospace;
        font-size: 12px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        z-index: 5;
        min-width: 140px;

        .coord-label {
            display: flex;
            justify-content: space-between;
            margin: 2px 0;
            color: #333;
            font-weight: 500;

            .coord-value {
                margin-left: 10px;
                color: #1976d2;
                font-weight: 600;
            }
        }
    }
}

</style>
