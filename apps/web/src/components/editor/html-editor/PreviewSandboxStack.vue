<script setup lang="ts">
/**
 * PreviewSandboxStack - 预览和沙盒垂直堆叠容器
 * 预览区域支持元素编辑，沙盒区域用于查看AI修改效果
 */
import { Check, X } from 'lucide-vue-next'
import { toast } from 'vue-sonner'
import { Button } from '@/components/ui/button'
import HtmlElementEditBar from './HtmlElementEditBar.vue'
import HtmlRenderBase from './HtmlRenderBase.vue'
import { useHtmlEditorStore } from './useHtmlEditorStore'
import { useHtmlSandboxStore } from './useHtmlSandboxStore'

const htmlEditorStore = useHtmlEditorStore()
const htmlSandboxStore = useHtmlSandboxStore()

const { htmlContent } = storeToRefs(htmlEditorStore)
const { sandboxContent, modifiedSections, isActive: isSandboxActive } = storeToRefs(htmlSandboxStore)

// 滚动同步状态
const sharedScrollTop = ref(0)

function handleScroll(scrollTop: number) {
  sharedScrollTop.value = scrollTop
}

function handleApply() {
  htmlSandboxStore.applySandboxToEditor((content: string) => {
    htmlEditorStore.setHtmlContent(content)
    if (htmlEditorStore.htmlEditor) {
      htmlEditorStore.htmlEditor.dispatch({
        changes: {
          from: 0,
          to: htmlEditorStore.htmlEditor.state.doc.length,
          insert: content,
        },
      })
    }
  })
  toast.success(`已应用 Sandbox 修改到编辑器`)
}

// 丢弃 Sandbox 内容
function handleDiscard() {
  htmlSandboxStore.closeSandbox()
  toast.info(`已丢弃 Sandbox 修改`)
}
</script>

<template>
  <div class="preview-sandbox-row" :class="{ 'has-sandbox': isSandboxActive }">
    <!-- 预览区域 - 支持元素编辑 -->
    <div class="row-panel preview-panel">
      <div class="panel-header">
        <span class="panel-title">预览（可编辑）</span>
      </div>
      <div class="panel-content">
        <HtmlElementEditBar
          :html-content="htmlContent"
          :scroll-top="sharedScrollTop"
          @scroll="handleScroll"
        />
      </div>
    </div>

    <div v-if="isSandboxActive" class="row-panel sandbox-panel">
      <div class="panel-header sandbox-header">
        <div class="header-left">
          <span class="panel-title">Sandbox 预览</span>
          <span
            v-if="modifiedSections.length > 0"
            class="modified-badge"
          >
            {{ modifiedSections.length }} 处修改
          </span>
        </div>
        <div class="header-actions">
          <Button
            variant="ghost"
            size="sm"
            class="action-btn apply-btn"
            title="应用修改"
            @click="handleApply"
          >
            <Check class="h-4 w-4 mr-1" />
            应用
          </Button>
          <Button
            variant="ghost"
            size="sm"
            class="action-btn discard-btn"
            title="丢弃修改"
            @click="handleDiscard"
          >
            <X class="h-4 w-4 mr-1" />
            丢弃
          </Button>
        </div>
      </div>
      <div class="panel-content">
        <HtmlRenderBase
          :html-content="sandboxContent"
          :scroll-top="sharedScrollTop"
          show-modified-markers
          @scroll="handleScroll"
        />
      </div>
    </div>
  </div>
</template>

<style scoped>
.preview-sandbox-row {
  display: flex;
  flex-direction: row;
  gap: 12px;
  width: 100%;
  height: 100%;
  padding: 12px;
  overflow: hidden;
}

.row-panel {
  display: flex;
  flex-direction: column;
  background: hsl(var(--card));
  border: 1px solid hsl(var(--border));
  border-radius: 12px;
  overflow: hidden;
  box-shadow:
    0 2px 4px -1px rgb(0 0 0 / 0.06),
    0 1px 2px -1px rgb(0 0 0 / 0.04);
  transition: box-shadow 0.2s ease;
}

.row-panel:hover {
  box-shadow:
    0 4px 8px -2px rgb(0 0 0 / 0.08),
    0 2px 4px -2px rgb(0 0 0 / 0.06);
}

.dark .row-panel {
  box-shadow:
    0 2px 4px -1px rgb(0 0 0 / 0.2),
    0 1px 2px -1px rgb(0 0 0 / 0.15);
}

/* 面板宽度 - 统一使用手机宽度以便对比 */
.preview-panel,
.sandbox-panel {
  width: 375px; /* iPhone 标准宽度 */
  max-width: 375px;
  flex-shrink: 0;
}

.preview-panel {
  margin: 0 auto; /* 仅在非沙盒模式下生效或由 flex 容器控制 */
}

/* 当没有沙盒时，预览面板居中 */
.preview-sandbox-row:not(.has-sandbox) {
  justify-content: center;
}

/* 当有沙盒时，两面板并排居中 */
.preview-sandbox-row.has-sandbox {
  justify-content: center;
}

/* 面板头部 */
.panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  border-bottom: 1px solid hsl(var(--border) / 0.5);
  background: hsl(var(--muted) / 0.3);
  min-height: 44px;
}

.panel-title {
  font-size: 13px;
  font-weight: 600;
  color: hsl(var(--muted-foreground));
}

.panel-content {
  flex: 1;
  min-height: 0;
  overflow: hidden;
}

.sandbox-iframe {
  width: 100%;
  height: 100%;
  border: none;
  background: hsl(var(--background));
  display: block;
}

/* 沙盒特殊样式 */
.sandbox-panel {
  border-color: hsl(45 93% 47% / 0.3);
}

.sandbox-header {
  background: linear-gradient(135deg, rgba(251, 191, 36, 0.1), rgba(245, 158, 11, 0.05));
}

.header-left {
  display: flex;
  align-items: center;
  gap: 10px;
}

.modified-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 10px;
  background: linear-gradient(135deg, #fbbf24, #f59e0b);
  color: white;
  font-weight: 500;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 4px;
}

.action-btn {
  height: 28px;
  padding: 0 10px;
  font-size: 12px;
  font-weight: 500;
  transition: all 0.2s ease;
}

.apply-btn {
  color: #16a34a;
}

.apply-btn:hover {
  background: rgba(22, 163, 74, 0.1);
  color: #15803d;
}

.discard-btn {
  color: #dc2626;
}

.discard-btn:hover {
  background: rgba(220, 38, 38, 0.1);
  color: #b91c1c;
}

/* 深色模式 */
.dark .sandbox-header {
  background: linear-gradient(135deg, rgba(251, 191, 36, 0.15), rgba(245, 158, 11, 0.1));
}
</style>
