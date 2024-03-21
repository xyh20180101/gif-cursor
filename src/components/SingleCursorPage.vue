<script setup>
import { h } from "vue"
import { NUpload, NUploadDragger, NIcon, NText, NP, NInput, NSpace, NImage, NButton, NTooltip } from 'naive-ui'
import { FileTray, Download, InformationCircle } from "@vicons/ionicons5"
import { parseGIF, decompressFrames } from 'gifuct-js'
import '../jszip.min.js'
</script>

<template>
    <n-upload :max="1" accept="image/gif" :custom-request="upload" :on-remove="remove">
        <n-upload-dragger>
            <div style="margin-bottom: 12px">
                <n-icon size="48" :depth="3">
                    <FileTray />
                </n-icon>
            </div>
            <n-text style="font-size: 16px">
                点击或者拖动文件到该区域来上传( *.gif )
            </n-text>
            <n-p depth="3" style="margin: 8px 0 0 0">
                文件仅在浏览器端处理，不会上传到服务器
            </n-p>
        </n-upload-dragger>
    </n-upload>

    <div class="t">帧列表</div>

    <n-space>
        <n-image v-for="c in frameCanvases" object-fit="contain" :width="c.width > 64 ? 64 : c.width"
            :height="c.height > 64 ? 64 : c.height" :src="c.toDataURL()" />
    </n-space>

    <div class="t">动画时长(ms)</div>
    <n-space>
        <n-input :disabled="!isUpload" type="text" :allow-input="onlyAllowNumber" placeholder=""
            v-model:value="cursorDuration" @input="() => { generateCss(); setPreview(); }" />
    </n-space>

    <div class="t" style="display:flex;align-items:center">
        <span>热点坐标(X, Y)</span>
        <n-tooltip>
            <template #trigger>
                <n-icon size="16">
                    <InformationCircle />
                </n-icon>
            </template>
            与 cur 指针中的热点概念相同（即指针的尖端），对于热点不在(0,0)的指针你需要手动设置
        </n-tooltip>
    </div>

    <n-space>
        <n-input :disabled="!isUpload" type="text" :allow-input="onlyAllowNumber" placeholder=""
            v-model:value="cursorHotpoint[0]" @input="() => { generateCss(); setPreview(); }" />
        <n-input :disabled="!isUpload" type="text" :allow-input="onlyAllowNumber" placeholder=""
            v-model:value="cursorHotpoint[1]" @input="() => { generateCss(); setPreview(); }" />
    </n-space>

    <div class="t">css</div>

    <n-input :disabled="!isUpload" :autosize="{ minRows: 15, maxRows: 15 }" v-model:value="cssText" type="textarea"
        readonly />

    <div class="t">预览区域</div>

    <div id="preview" style="width: 100%; height: 50px;border-radius:3px;border: 1px solid rgb(224, 224, 230);"></div>

    <n-button :disabled="!isUpload" style="margin:8px 0 4px 0;width: 100%;" secondary strong @click="download">
        <template #icon>
            <n-icon>
                <Download />
            </n-icon>
        </template>
        打包下载（png、css）
    </n-button>
</template>

<script>
const onlyAllowNumber = (value) => !value || /^\d+$/.test(value)

export default {
    data() {
        return {
            isUpload: false,
            file: null,
            cursorName: '',
            cursorType: 'default',
            cursorHotpoint: [0, 0],
            cursorDuration: 0,
            cursorTypeOptions: [
                {
                    type: 'group', label: '通常', key: 'general', children: [
                        { label: 'default', value: 'default' }
                    ]
                },
                {
                    type: 'group', label: '链接及状态', key: 'link', children: [
                        { label: 'context-menu', value: 'context-menu' },
                        { label: 'help', value: 'help' },
                        { label: 'pointer', value: 'pointer' },
                        { label: 'progress', value: 'progress' },
                        { label: 'wait', value: 'wait' }
                    ]
                },
                {
                    type: 'group', label: '选择', key: 'selection', children: [
                        { label: 'cell', value: 'cell' },
                        { label: 'crosshair', value: 'crosshair' },
                        { label: 'text', value: 'text' },
                        { label: 'vertical-text', value: 'vertical-text' }
                    ]
                },
                {
                    type: 'group', label: '拖拽', key: 'drag', children: [
                        { label: 'alias', value: 'alias' },
                        { label: 'copy', value: 'copy' },
                        { label: 'move', value: 'move' },
                        { label: 'no-drop', value: 'no-drop' },
                        { label: 'not-allowed', value: 'not-allowed' },
                        { label: 'grab', value: 'grab' },
                        { label: 'grabbing', value: 'grabbing' }
                    ]
                },
                {
                    type: 'group', label: '重设大小及滚动', key: 'scroll', children: [
                        { label: 'all-scroll', value: 'all-scroll' },
                        { label: 'col-resize', value: 'col-resize' },
                        { label: 'row-resize', value: 'row-resize' },
                        { label: 'n-resize', value: 'n-resize' },
                        { label: 'e-resize', value: 'e-resize' },
                        { label: 's-resize', value: 's-resize' },
                        { label: 'w-resize', value: 'w-resize' },
                        { label: 'ne-resize', value: 'ne-resize' },
                        { label: 'nw-resize', value: 'nw-resize' },
                        { label: 'se-resize', value: 'se-resize' },
                        { label: 'sw-resize', value: 'sw-resize' },
                        { label: 'ew-resize', value: 'ew-resize' },
                        { label: 'ns-resize', value: 'ns-resize' },
                        { label: 'nesw-resize', value: 'nesw-resize' },
                        { label: 'nwse-resize', value: 'nwse-resize' }
                    ]
                },
                {
                    type: 'group', label: '缩放', key: 'zoom', children: [
                        { label: 'zoom-in', value: 'zoom-in' },
                        { label: 'zoom-out', value: 'zoom-out' }
                    ]
                },
            ],
            renderLabel: (option) => {
                if (option.type === 'group') return option.label
                return [
                    h(
                        'div',
                        { style: { cursor: option.value, width: '100%', height: '100%', display: 'flex', 'align-items': 'center' } },
                        option.label
                    )
                ]
            },
            frames: [],
            frameCanvases: [],
            cssText: '',
            previewCssText: ''
        }
    },
    methods: {
        upload(o) {
            this.isUpload = true
            this.file = o.file
            this.processGif(o.file)
        },

        remove() {
            this.isUpload = false
            this.cursorName = ''
            this.cursorType = 'default'
            this.cursorDuration = 0
            this.cursorHotpoint = [0, 0]
            this.frames = []
            this.frameCanvases = []
            this.cssText = ''
            this.previewCssText = ''
        },

        processGif(file) {
            const reader = new FileReader()

            var _this = this

            reader.addEventListener('load', (f) => {
                const arrayBuffer = f.target.result
                const gif = parseGIF(arrayBuffer)
                _this.frames = decompressFrames(gif, true)
                _this.frames.forEach(function (frame) {
                    _this.frameCanvases.push(_this.getCanvasFromFrame(frame))
                })

                _this.cursorName = _this.removeFileExtension(_this.file.name)
                _this.cursorDuration = _this.frames[0].delay * _this.frames.length
                _this.generateCss()
                _this.setPreview()
            })

            reader.readAsArrayBuffer(file.file)
        },

        getCanvasFromFrame(frame) {
            const canvas = document.createElement('canvas')
            const ctx = canvas.getContext('2d')

            canvas.width = frame.dims.width
            canvas.height = frame.dims.height

            const imageData = ctx.createImageData(canvas.width, canvas.height)
            imageData.data.set(frame.patch)

            ctx.putImageData(imageData, 0, 0)

            return canvas
        },

        generateCss() {
            //指针名称
            const cursorName = this.cursorName

            //指针样式
            const cursorType = this.cursorType

            //指针动画时长
            const cursorDuration = this.cursorDuration

            //热点坐标
            const cursorHotpoint = this.cursorHotpoint

            //动画步长百分比
            const step = 100 / (this.frames.length - 1)

            //以下是为用户生成的css

            this.cssText = `*{cursor: url(${cursorName}_0.png) ${cursorHotpoint[0]} ${cursorHotpoint[1]}, ${cursorType};-webkit-animation: cursor_${cursorName} ${cursorDuration}ms infinite;animation: cursor_${cursorName} ${cursorDuration}ms infinite;}`

            let keyframesText = ''

            for (var i = 0; i < this.frames.length; i++) {
                keyframesText += `${(step * i).toFixed(2)}%{cursor:url(${cursorName}_${i}.png) ${cursorHotpoint[0]} ${cursorHotpoint[1]}, ${cursorType};}`
            }

            this.cssText += `@-webkit-keyframes cursor_${cursorName} {${keyframesText}} @keyframes cursor_${cursorName} {${keyframesText}}`

            this.cssText = this.formatCSS(this.cssText)

            //以下是为预览生成的css,这个css会被应用到预览区域

            this.previewCssText = `#preview{cursor: url(${this.frameCanvases[0].toDataURL('image/png')}) ${cursorHotpoint[0]} ${cursorHotpoint[1]},${cursorType};-webkit-animation: cursor_${cursorName} ${cursorDuration}ms infinite;animation: cursor_${cursorName} ${cursorDuration}ms infinite;}`

            let previewKeyframesText = ''

            for (var i = 0; i < this.frames.length; i++) {
                previewKeyframesText += `${(step * i).toFixed(2)}%{cursor:url(${this.frameCanvases[i].toDataURL('image/png')}) ${cursorHotpoint[0]} ${cursorHotpoint[1]}, ${cursorType};}`
            }

            this.previewCssText += `@-webkit-keyframes cursor_${cursorName} {${previewKeyframesText}} @keyframes cursor_${cursorName} {${previewKeyframesText}}`
        },

        setPreview() {
            const link = document.createElement('link')
            link.rel = 'stylesheet'
            link.type = 'text/css'

            const cssTextBase64 = btoa(this.previewCssText)
            const dataUrl = `data:text/css;base64,${cssTextBase64}`

            link.href = dataUrl
            document.head.appendChild(link)
        },

        download() {
            const zip = new JSZip()

            for (var i = 0; i < this.frameCanvases.length; i++) {
                const canvas = this.frameCanvases[i]
                const dataUrl = canvas.toDataURL('image/png')
                zip.file(`cursors/${this.cursorName}_${i}.png`, dataUrl.substr(dataUrl.indexOf(',') + 1), { base64: true })
            }

            zip.file('cursors/cursor.css', this.cssText)

            zip.generateAsync({ type: 'blob' })
                .then(blob => {
                    const downloadLink = document.createElement('a')
                    downloadLink.href = URL.createObjectURL(blob)
                    downloadLink.download = 'cursors.zip'

                    document.body.appendChild(downloadLink)
                    downloadLink.click()
                    document.body.removeChild(downloadLink)
                })
        },

        ///以下是辅助方法

        removeFileExtension(filename) {
            const lastDotIndex = filename.lastIndexOf('.')

            if (lastDotIndex !== -1 && lastDotIndex < filename.length - 1) {
                return filename.slice(0, lastDotIndex)
            }

            return filename
        },

        formatCSS(cssText) {
            cssText = cssText.replace(/\s+/g, ' ');

            cssText = cssText.replace(/\s*{\s*/g, ' {\n');

            cssText = cssText.replace(/\s*}\s*/g, '\n}\n');

            cssText = cssText.replace(/;\s*/g, ';\n');

            return cssText;
        }
    }
}
</script>

<style>
.t {
    margin: 8px 0 4px 0;
}

.n-base-select-option {
    align-items: stretch !important;
}

.n-base-select-option__content {
    width: 100% !important;
}
</style>