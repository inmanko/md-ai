<script setup lang="ts">
import { usePreviewStyleStore } from './usePreviewStyleStore'

/**
 * HtmlRenderBase - 底层公共渲染组件
 * 负责在 iframe 中一致地渲染 HTML 内容，支持滚动同步和样式隔离
 */
const props = defineProps<{
  htmlContent: string
  scrollTop?: number
  showModifiedMarkers?: boolean // 是否显示 AI 修改标记
}>()

const emit = defineEmits<{
  scroll: [scrollTop: number]
  load: [document: Document]
}>()

const iframeRef = ref<HTMLIFrameElement>()
const previewStyleStore = usePreviewStyleStore()
const STYLE_ELEMENT_ID = `preview-style-overrides`

// 生成样式文本
function generateStyleText() {
  try {
    const styleOverrides = previewStyleStore.styleOverrides
    return styleOverrides
      .map((override) => {
        const styles = Object.entries(override.styles)
          .map(([key, value]) => `  ${key.replace(/([A-Z])/g, `-$1`).toLowerCase()}: ${value} !important;`)
          .join(`\n`)
        return `${override.selector} {\n${styles}\n}`
      })
      .join(`\n\n`)
  }
  catch (e) {
    console.error(`Failed to generate style overrides:`, e)
    return ``
  }
}

// 动态更新 iframe 内部样式
function updateIframeStyles() {
  if (!iframeRef.value?.contentDocument)
    return

  const doc = iframeRef.value.contentDocument
  let styleElement = doc.getElementById(STYLE_ELEMENT_ID) as HTMLStyleElement

  if (!styleElement) {
    styleElement = doc.createElement(`style`)
    styleElement.id = STYLE_ELEMENT_ID
    doc.head.appendChild(styleElement)
  }

  styleElement.textContent = generateStyleText()
}

// 监听样式变化
watch(() => previewStyleStore.styleOverrides, () => {
  updateIframeStyles()
}, { deep: true })

// 生成完整的 HTML 文档
function generateFullHtml(content: string): string {
  const styleOverridesText = generateStyleText()

  return `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <style>
        * {
          margin: 0;
          padding: 0;
          box-sizing: border-box;
        }
        body {
          font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen', 'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue', sans-serif;
          padding: 16px;
          overflow-y: auto;
          background: transparent;
        }
        img {
          max-width: 100%;
          height: auto;
          border-radius: 8px;
        }
        pre {
          overflow-x: auto;
          padding: 1em;
          background: #f5f5f5;
          border-radius: 8px;
        }
        code {
          font-family: 'JetBrains Mono', 'Fira Code', 'Consolas', 'Monaco', 'Courier New', monospace;
        }
        
        /* 样式覆盖注入 */
        ${styleOverridesText}

        /* 修改标记样式 (仅在 showModifiedMarkers 为 true 时生效) */
        ${props.showModifiedMarkers
          ? `
        [data-sandbox-modified] {
          position: relative;
          outline: 2px dashed rgba(251, 191, 36, 0.8) !important;
          outline-offset: 4px;
          background: linear-gradient(135deg, rgba(251, 191, 36, 0.1) 0%, rgba(245, 158, 11, 0.05) 100%) !important;
          border-radius: 8px;
          animation: sandboxPulse 2s ease-in-out infinite;
        }
        [data-sandbox-modified]::before {
          content: '已修改';
          position: absolute;
          top: -10px;
          right: -10px;
          font-size: 10px;
          padding: 3px 8px;
          background: linear-gradient(135deg, #fbbf24, #f59e0b);
          color: white;
          border-radius: 6px;
          font-weight: 600;
          box-shadow: 0 2px 8px rgba(251, 191, 36, 0.3);
          z-index: 10;
        }
        @keyframes sandboxPulse {
          0%, 100% {
            outline-color: rgba(251, 191, 36, 0.6);
            box-shadow: 0 0 0 0 rgba(251, 191, 36, 0.2);
          }
          50% {
            outline-color: rgba(251, 191, 36, 1);
            box-shadow: 0 0 0 4px rgba(251, 191, 36, 0.1);
          }
        }
        `
          : ``}

        /* 隐藏滚动条但保留滚动功能 */
        body::-webkit-scrollbar {
          display: none;
        }
        body {
          -ms-overflow-style: none;
          scrollbar-width: none;
        }
      </style>
    </head>
    <body id="html-output">
      ${content}
    </body>
    </html>
  `
}

const srcdoc = computed(() => generateFullHtml(props.htmlContent))

// 处理 iframe 加载完成
function handleLoad() {
  if (!iframeRef.value?.contentDocument)
    return

  const doc = iframeRef.value.contentDocument

  // 初始应用样式
  updateIframeStyles()

  emit(`load`, doc)

  // 监听滚动
  doc.addEventListener(`scroll`, () => {
    emit(`scroll`, doc.documentElement.scrollTop || doc.body.scrollTop)
  })

  // 同步初始滚动位置
  if (props.scrollTop !== undefined) {
    doc.documentElement.scrollTop = props.scrollTop
    doc.body.scrollTop = props.scrollTop
  }
}

// 监听外部滚动同步请求
watch(() => props.scrollTop, (newVal) => {
  if (!iframeRef.value?.contentDocument || newVal === undefined)
    return
  const doc = iframeRef.value.contentDocument
  const currentScroll = doc.documentElement.scrollTop || doc.body.scrollTop
  if (Math.abs(currentScroll - newVal) > 2) {
    doc.documentElement.scrollTop = newVal
    doc.body.scrollTop = newVal
  }
})

// 暴露 iframe 给父组件
defineExpose({
  getIframe: () => iframeRef.value,
  getDocument: () => iframeRef.value?.contentDocument,
})
</script>

<template>
  <div class="html-render-base h-full w-full">
    <iframe
      ref="iframeRef"
      class="render-iframe h-full w-full border-none"
      :srcdoc="srcdoc"
      sandbox="allow-same-origin allow-scripts"
      @load="handleLoad"
    />
  </div>
</template>

<style scoped>
.html-render-base {
  overflow: hidden;
  background: hsl(var(--background));
}
.render-iframe {
  display: block;
}
</style>
