<script setup>
import { h, ref } from "vue"
import { NUpload, NUploadDragger, NIcon, NText, NP, NInput, NSpace, NImage, NButton, NSelect, NDataTable, NGrid, NGi } from 'naive-ui'
import { FileTray, Download } from "@vicons/ionicons5"
import { parseGIF, decompressFrames } from 'gifuct-js'
import '../jszip.min.js'
</script>

<template>
    <n-upload ref="upload" :max="34" multiple accept="image/gif" v-model:file-list="fileList" :show-file-list="false"
        :custom-request="upload" :on-remove="remove">
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

    <n-data-table style="margin-top:8px" :columns="columns" :data="cursorList" :row-key="() => 'cursorName'" />

    <div style="width:100%;display:flex;justify-content:right">
        <n-button :disabled="!isUpload" style="margin:8px 0 4px 0" type="error" strong @click="remove">清空</n-button>
    </div>

    <n-grid x-gap="12" :cols="2">
        <n-gi>
            <div style="margin:8px 0 4px 0">css</div>
            <n-input :disabled="!isUpload" :autosize="{ minRows: 15, maxRows: 15 }" v-model:value="cssText" type="textarea"
                readonly placeholder="" />
        </n-gi>
        <n-gi>
            <div style="margin:8px 0 4px 0">js</div>
            <n-input :disabled="!isUpload" :autosize="{ minRows: 15, maxRows: 15 }" v-model:value="jsText" type="textarea"
                readonly placeholder="" />
        </n-gi>
    </n-grid>

    <div style="margin:8px 0 4px 0">预览区域</div>

    <div id="preview">
        <n-space style="gap:0" v-for="g in cursorTypeOptions">
            <n-button :class="'cursor-' + t.value" style="margin:4px" v-for="t in g.children">{{ t.label }}</n-button>
        </n-space>
    </div>

    <n-button :disabled="!isUpload" style="margin:8px 0 4px 0;width: 100%;" secondary strong @click="download">
        <template #icon>
            <n-icon>
                <Download />
            </n-icon>
        </template>
        打包下载（png拆分和样式文件）
    </n-button>
</template>

<script>
var cursorTypeOptions = [
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
]
const renderLabel = (option) => {
    if (option.type === 'group') return option.label
    return [
        h(
            'div',
            { style: { cursor: option.value, width: '100%', height: '100%', display: 'flex', 'align-items': 'center' } },
            option.label
        )
    ]
}

var observer = new MutationObserver(() => { })

export default {
    data() {
        var _this = this
        return {
            isUpload: false,
            fileList: [],
            cursorList: [],
            columns: [
                { title: '指针名称', key: 'cursorName' },
                {
                    title: '帧列表', key: 'frameCanvases', render(row) {
                        return h(NSpace, {}, row.frameCanvases.map((f) => {
                            return h(NImage, {
                                width: f.width,
                                height: f.height,
                                maxwidth: 100,
                                src: f.toDataURL()
                            }, {})
                        }))
                    }
                },
                {
                    title: '指针类型', key: 'cursorType', render(row) {
                        return h(NSelect,
                            {
                                value: row.cursorType,
                                options: cursorTypeOptions,
                                renderLabel: renderLabel,
                                onUpdateValue: (v) => { row.cursorType = v; _this.generateCss(); _this.generateJs(); _this.setPreview(); }
                            }, {})
                    }
                },
                {
                    title: '动画时长', key: 'cursorDuration', render(row) {
                        return h(NInput,
                            {
                                value: row.cursorDuration,
                                type: 'text',
                                allowInput: 'onlyAllowNumber',
                                placeholder: '只能输入数字',
                                onInput: (v) => { row.cursorDuration = v; _this.generateCss(); _this.generateJs(); _this.setPreview(); }
                            }, {})
                    }
                },
            ],
            cssText: '',
            jsText: '',
            previewCssText: '',
            previewJsText: ''
        }
    },
    methods: {
        upload(o) {
            if (o.file.status === 'pending') {
                this.isUpload = true
                this.processGif(o.file)
                o.file.status = 'finished'
            }
        },

        remove() {
            this.isUpload = false
            this.fileList = []
            this.cursorList = []
            this.cssText = ''
            this.previewCssText = ''
            this.jsText = ''
            this.previewJsText = ''
        },

        processGif(file) {
            const reader = new FileReader()

            var _this = this

            reader.addEventListener('load', (f) => {
                const arrayBuffer = f.target.result
                const gif = parseGIF(arrayBuffer)
                let cursor = {
                    cursorName: '',
                    cursorType: 'default',
                    cursorDuration: 0,
                    frames: [],
                    frameCanvases: []
                }
                cursor.frames = decompressFrames(gif, true)
                cursor.frames.forEach(function (frame) {
                    cursor.frameCanvases.push(_this.getCanvasFromFrame(frame))
                })

                cursor.cursorName = _this.removeFileExtension(file.name)
                cursor.cursorDuration = cursor.frames[0].delay * cursor.frames.length
                _this.cursorList.push(cursor)
                _this.generateCss()
                _this.generateJs()
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
            this.cssText = ''
            this.previewCssText = ''

            for (let i = 0; i < this.cursorList.length; i++) {
                const cursor = this.cursorList[i]

                //指针名称
                const cursorName = cursor.cursorName

                //指针样式
                const cursorType = cursor.cursorType
                console.log(cursorType)

                //指针动画时长
                const cursorDuration = cursor.cursorDuration

                //动画步长百分比
                const step = 100 / (cursor.frames.length - 1)

                //以下是为用户生成的css

                let keyframesText = ''

                for (let i = 0; i < cursor.frames.length; i++) {
                    keyframesText += `${(step * i).toFixed(2)}%{cursor:url(${cursorName}_${i}.png),${cursorType};}`
                }

                this.cssText += `@-webkit-keyframes cursor_${cursorName} {${keyframesText}} @keyframes cursor_${cursorName} {${keyframesText}}`

                //以下是为预览生成的css,这个css会被应用到预览区域

                this.previewCssText += `#preview{cursor: url(${cursor.frameCanvases[0].toDataURL('image/png')}),${cursorType};-webkit-animation: cursor_${cursorName} ${cursorDuration}ms infinite;animation: cursor_${cursorName} ${cursorDuration}ms infinite;}`

                let previewKeyframesText = ''

                for (let i = 0; i < cursor.frames.length; i++) {
                    previewKeyframesText += `${(step * i).toFixed(2)}%{cursor:url(${cursor.frameCanvases[i].toDataURL('image/png')}),${cursorType};}`
                }

                this.previewCssText += `@-webkit-keyframes cursor_${cursorName} {${previewKeyframesText}} @keyframes cursor_${cursorName} {${previewKeyframesText}}`
            }

            this.cssText = this.formatCSS(this.cssText)
        },

        generateJs() {
            this.jsText = `var observer = new MutationObserver(function (mutationsList, observer) {
    for (const mutation of mutationsList) {
        if (mutation.type === 'childList') {
            mutation.addedNodes.forEach(function (node) {
                if (node instanceof Element) {
                    const computedStyle = window.getComputedStyle(node)
                    const cursorType = computedStyle.getPropertyValue('cursor')
                    switch (cursorType) {`
            this.previewJsText = this.jsText

            for (let i = 0; i < this.cursorList.length; i++) {
                const cursor = this.cursorList[i]

                //指针名称
                const cursorName = cursor.cursorName

                //指针样式
                const cursorType = cursor.cursorType

                //指针动画时长
                const cursorDuration = cursor.cursorDuration

                //动画步长百分比
                const step = 100 / (cursor.frames.length - 1)

                //以下是为用户生成的js

                this.jsText += `
                        case '${cursorType}':
                                node.style.cursor = 'url(${cursorName}_0.png)'
                                node.style.animation = 'cursor_${cursorName} ${cursorDuration}ms infinite'
                                node.style.webkitAnimation = 'cursor_${cursorName} ${cursorDuration}ms infinite'
                            break`

                //以下是为预览生成的js,这个js会被应用到预览区域

                this.previewJsText += `
                        case '${cursorType}':
                                node.style.cursor = 'url(${cursor.frameCanvases[0].toDataURL('image/png')})'
                                node.style.animation = 'cursor_${cursorName} ${cursorDuration}ms infinite'
                                node.style.webkitAnimation = 'cursor_${cursorName} ${cursorDuration}ms infinite'
                            break`
            }

            const lastJs = `
                        default:
                            break
                    }
                }
            })
        }
    }
})
observer.observe(document.body, { childList: true, subtree: true })`
            this.jsText += lastJs
            this.previewJsText += lastJs
        },

        setPreview() {
            const link = document.createElement('link')
            link.rel = 'stylesheet'
            link.type = 'text/css'

            const cssTextBase64 = btoa(this.previewCssText)
            const dataUrl = `data:text/css;base64,${cssTextBase64}`

            link.href = dataUrl
            document.head.appendChild(link)

            observer.disconnect()
            eval(this.previewJsText)
        },

        download() {
            const zip = new JSZip()
            for (var i = 0; i < this.cursorList.length; i++) {
                for (var j = 0; j < this.cursorList[i].frameCanvases.length; j++) {
                    const canvas = this.cursorList[i].frameCanvases[j]
                    const dataUrl = canvas.toDataURL('image/png')
                    zip.file(`${this.cursorList[i].cursorName}_${j}.png`, dataUrl.substr(dataUrl.indexOf(',') + 1), { base64: true })
                }
            }


            zip.file('cursor.css', this.cssText)
            zip.file('cursor.js', this.jsText)

            zip.generateAsync({ type: 'blob' })
                .then(blob => {
                    // 创建一个下载链接
                    const downloadLink = document.createElement('a')
                    downloadLink.href = URL.createObjectURL(blob)
                    downloadLink.download = 'cursors.zip'

                    // 模拟点击触发下载
                    document.body.appendChild(downloadLink)
                    downloadLink.click()
                    document.body.removeChild(downloadLink)
                });
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
.n-base-select-option {
    align-items: stretch !important;
}

.n-base-select-option__content {
    width: 100% !important;
}

.cursor-default {
    cursor: default;
}

.cursor-context-menu {
    cursor: context-menu;
}

.cursor-help {
    cursor: help;
}

.cursor-pointer {
    cursor: pointer;
}

.cursor-progress {
    cursor: progress;
}

.cursor-wait {
    cursor: wait;
}

.cursor-cell {
    cursor: cell;
}

.cursor-crosshair {
    cursor: crosshair;
}

.cursor-vertical-text {
    cursor: vertical-text;
}

.cursor-alias {
    cursor: alias;
}

.cursor-copy {
    cursor: copy;
}

.cursor-move {
    cursor: move;
}

.cursor-no-drop {
    cursor: no-drop;
}

.cursor-not-allowed {
    cursor: not-allowed;
}

.cursor-grab {
    cursor: grab;
}

.cursor-grabbing {
    cursor: grabbing;
}

.cursor-all-scroll {
    cursor: all-scroll;
}

.cursor-col-resize {
    cursor: col-resize;
}

.cursor-row-resize {
    cursor: row-resize;
}

.cursor-n-resize {
    cursor: n-resize;
}

.cursor-e-resize {
    cursor: e-resize;
}

.cursor-s-resize {
    cursor: s-resize;
}

.cursor-w-resize {
    cursor: w-resize;
}

.cursor-ne-resize {
    cursor: ne-resize;
}

.cursor-nw-resize {
    cursor: nw-resize;
}

.cursor-se-resize {
    cursor: se-resize;
}

.cursor-sw-resize {
    cursor: sw-resize;
}

.cursor-ew-resize {
    cursor: ew-resize;
}

.cursor-ns-resize {
    cursor: ns-resize;
}

.cursor-nesw-resize {
    cursor: nesw-resize;
}

.cursor-nwse-resize {
    cursor: nwse-resize;
}

.cursor-zoom-in {
    cursor: zoom-in;
}

.cursor-zoom-out {
    cursor: zoom-out;
}
</style>