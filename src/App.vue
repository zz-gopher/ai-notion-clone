<script setup>
import { ref, nextTick, computed, watch } from 'vue'
import { marked } from 'marked' // 💡 新增：引入 Markdown 解析库

// ==========================================
// 1. 核心状态数据
// ==========================================
const pageTitle = ref('AI 智能笔记')

const blocks = ref([
  { id: '1', type: 'p', content: '' }
])

// 💡 新增：记录当前正在编辑的积木块 ID
const activeBlockId = ref(null) 

const wordCount = computed(() => {
  let count = pageTitle.value.length
  blocks.value.forEach(block => {
    count += block.content.length
  })
  return count
})

const lastEditedTime = ref('刚刚')

watch([pageTitle, blocks], () => {
  const now = new Date()
  const hours = now.getHours().toString().padStart(2, '0')
  const minutes = now.getMinutes().toString().padStart(2, '0')
  lastEditedTime.value = `${hours}:${minutes}`
}, { deep: true })

const inputRefs = ref([])

// ==========================================
// 💡 新增：双态切换核心逻辑
// ==========================================
const renderMarkdown = (text, type) => {
  if (!text) {
    // 模拟占位符
    const placeholderText = type === 'h1' ? '无标题' : '输入 \'/\' 唤出菜单...'
    return `<span class="placeholder-text">${placeholderText}</span>`
  }
  // 渲染为 HTML
  return marked.parse(text)
}

const focusBlock = (id, index) => {
  activeBlockId.value = id
  nextTick(() => {
    const textarea = inputRefs.value[index]
    if (textarea) {
      textarea.focus()
      // 恢复高度自适应
      textarea.style.height = 'auto'
      textarea.style.height = textarea.scrollHeight + 'px'
    }
  })
}

// ==========================================
// 2. 基础交互逻辑：回车与退格
// ==========================================
const handleEnter = async (index) => {
  if (showMenu.value) {
    const selectedOption = menuOptions[selectedMenuIndex.value]
    turnInto(selectedOption.type)
    return
  }

  const newBlockId = Date.now().toString()
  const newBlock = { id: newBlockId, type: 'p', content: '' }
  blocks.value.splice(index + 1, 0, newBlock)
  
  // 💡 关键：新建行必须立刻进入编辑态，否则光标无法聚焦
  activeBlockId.value = newBlockId 

  await nextTick()
  if (inputRefs.value[index + 1]) {
    inputRefs.value[index + 1].focus()
  }
}

const handleBackspace = async (index, event) => {
  const currentBlock = blocks.value[index]
  if (currentBlock.content === '' && index > 0) {
    event.preventDefault()
    blocks.value.splice(index, 1)

    await nextTick()
    // 退格到上一行时，上一行也必须进入编辑态
    activeBlockId.value = blocks.value[index - 1].id
    if (inputRefs.value[index - 1]) {
      inputRefs.value[index - 1].focus()
    }
  }
}

const triggerAI = async (index) => {
  blocks.value[index].content = blocks.value[index].content.slice(0, -1)
  const prompt = blocks.value[index].content
  
  const aiBlockId = Date.now().toString()
  const aiBlock = { id: aiBlockId, type: 'p', content: '' }
  blocks.value.splice(index + 1, 0, aiBlock)
  
  // 💡 注意：这里不把 aiBlockId 设为 activeBlockId。
  // 这样 AI 在吐字时，页面上会直接渲染出加粗的 Markdown 效果，极度极客！

  try {
    const response = await fetch('http://localhost:8080/api/ai/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ prompt: prompt })
    })

    const reader = response.body.getReader()
    const decoder = new TextDecoder('utf-8')

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      const chunk = decoder.decode(value)
      const lines = chunk.split('\n')
      
      for (const line of lines) {
        if (line.startsWith('data:')) {
          const dataStr = line.substring(5).trim()
          if (dataStr === '[DONE]') continue
          try {
            const data = JSON.parse(dataStr)
            const textContent = data.choices[0].delta.content || ''
            
            const targetBlock = blocks.value.find(b => b.id === aiBlockId)
            if (targetBlock) {
              targetBlock.content += textContent
              // 因为是渲染态(div)，DOM 会根据文字多少自然撑开高度，不再需要我们手动计算高度了！
            }
          } catch (e) {}
        }
      }
    }
  } catch (error) {
    console.error('AI 请求失败:', error)
  }
}

// ==========================================
// 3. 高级交互逻辑：斜杠 / 菜单
// ==========================================
const showMenu = ref(false)
const menuPosition = ref({ top: '0px', left: '0px' })
const activeBlockIndex = ref(null)

const menuOptions = [
  { type: 'ai', label: '✨ AI 帮我续写' },
  { type: 'h1', label: '一级标题 (H1)' },
  { type: 'h2', label: '二级标题 (H2)' },
  { type: 'p', label: '普通段落 (Text)' }
]
const selectedMenuIndex = ref(0)

const handleInput = (index, event) => {
  const textarea = event.target
  textarea.style.height = 'auto'
  textarea.style.height = textarea.scrollHeight + 'px'

  const currentBlock = blocks.value[index]
  if (currentBlock.content.endsWith('/')) {
    const rect = textarea.getBoundingClientRect()
    menuPosition.value = {
      top: `${rect.bottom + window.scrollY + 5}px`,
      left: `${rect.left + window.scrollX + 20}px`
    }
    showMenu.value = true
    activeBlockIndex.value = index
    selectedMenuIndex.value = 0
  } else {
    showMenu.value = false
  }
}

const turnInto = (newType) => {
  if (activeBlockIndex.value !== null) {
    if (newType === 'ai') {
      triggerAI(activeBlockIndex.value)
    } else {
      const block = blocks.value[activeBlockIndex.value]
      block.type = newType
      block.content = block.content.slice(0, -1) 
    }
    showMenu.value = false
    activeBlockIndex.value = null
  }
}

const handleArrowKey = (direction, event) => {
  if (!showMenu.value) return
  event.preventDefault()
  if (direction === 'up') {
    selectedMenuIndex.value = (selectedMenuIndex.value - 1 + menuOptions.length) % menuOptions.length
  } else if (direction === 'down') {
    selectedMenuIndex.value = (selectedMenuIndex.value + 1) % menuOptions.length
  }
}
</script>

<template>
  <div class="editor-container">

    <input type="text" v-model="pageTitle" class="page-title" placeholder="无标题">

    <TransitionGroup name="list" tag="div" class="blocks-wrapper">
      <div v-for="(block, index) in blocks" :key="block.id" class="block-item">
        <div class="block-handle">
          <span class="drag-icon">⋮⋮</span>
        </div>

        <!-- 💡 状态 A：渲染态 -->
        <div 
          v-if="activeBlockId !== block.id"
          class="markdown-body block-input" 
          :class="[`block-${block.type}`]"
          v-html="renderMarkdown(block.content, block.type)"
          @click="focusBlock(block.id, index)"
        ></div>

        <!-- 💡 状态 B：编辑态 -->
        <textarea 
          v-else
          v-model="block.content" 
          :class="['block-input', `block-${block.type}`]"
          :placeholder="block.type === 'h1' ? '无标题' : '输入 \'/\' 唤出菜单...'" 
          rows="1"
          @keydown.enter.prevent="handleEnter(index)" 
          @keydown.delete="handleBackspace(index, $event)"
          @keydown.up="handleArrowKey('up', $event)" 
          @keydown.down="handleArrowKey('down', $event)"
          @input="handleInput(index, $event)" 
          @blur="activeBlockId = null" 
          :ref="(el) => inputRefs[index] = el"
        ></textarea>
      </div>
    </TransitionGroup>

    <div v-show="showMenu" class="command-menu" :style="{ top: menuPosition.top, left: menuPosition.left }">
      <div class="menu-title">转换类型</div>
      <div v-for="(item, i) in menuOptions" :key="item.type"
        :class="['menu-item', { 'is-selected': selectedMenuIndex === i }]" @click="turnInto(item.type)"
        @mouseenter="selectedMenuIndex = i">
        {{ item.label }}
      </div>
    </div>
    
    <div class="status-bar" v-show="wordCount > 0">
      最后编辑于 {{ lastEditedTime }} • {{ wordCount }} 字
    </div>
  </div>
</template>

<style scoped>
/* 原有基础排版保持不变 */
.editor-container {
  max-width: 700px;
  margin: 40px auto;
  padding: 0 40px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
  color: #37352f;
}

.page-title {
  width: 100%;
  border: none;
  outline: none;
  background: transparent;
  font-size: 2.5em;
  font-weight: 700;
  color: #37352f;
  margin-bottom: 20px;
  padding-left: 28px;
}

.page-title::placeholder {
  color: #e0e0e0;
}

.blocks-wrapper {
  position: relative;
}

.block-item {
  display: flex;
  align-items: flex-start;
  margin-bottom: 2px;
  position: relative;
  transition: all 0.2s ease;
}

.block-handle {
  width: 24px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-right: 4px;
  margin-top: 4px;
  opacity: 0;
  transition: opacity 0.2s;
  color: #c1c1bf;
  font-size: 14px;
  cursor: grab;
}

.block-item:hover .block-handle {
  opacity: 1;
}

.block-input {
  flex: 1;
  border: none;
  outline: none;
  background: transparent;
  padding: 4px 2px;
  width: 100%;
  resize: none; 
  overflow: hidden; 
  font-family: inherit; 
}

/* 💡 新增：统一 markdown-body 和 textarea 的基础体验 */
.markdown-body {
  cursor: text;
  min-height: 28px; /* 保证空行也能被点击到 */
}

/* 💡 新增：清除 marked 解析出来的默认段落外边距，防止排版抖动 */
.markdown-body :deep(p) {
  margin: 0;
  word-wrap: break-word;
}
.markdown-body :deep(strong) {
  font-weight: 700;
  color: #1a1a1a;
}
/* 占位符颜色 */
.placeholder-text {
  color: #e0e0e0;
  pointer-events: none;
}

textarea.block-input::placeholder {
  color: #e0e0e0;
}

textarea.block-input:focus {
  border-left: 2px solid #ebeced;
  padding-left: 10px;
  margin-left: -12px;
}

textarea.block-input::selection {
  background: rgba(46, 170, 220, 0.3);
}

.block-h1 {
  font-size: 2.2em;
  font-weight: 700;
  margin-top: 1.2em;
  margin-bottom: 0.2em;
}

.block-h2 {
  font-size: 1.6em;
  font-weight: 600;
  margin-top: 1em;
  margin-bottom: 0.1em;
}

.block-p {
  font-size: 1.05em;
  line-height: 1.6;
}

.list-enter-active,
.list-leave-active {
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
}
.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateY(15px);
}
.list-leave-active {
  position: absolute;
  width: calc(100% - 28px);
}

.command-menu {
  position: absolute;
  background: white;
  border: 1px solid #efefef;
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.08);
  border-radius: 8px;
  width: 220px;
  z-index: 100;
  padding: 6px;
}

.menu-title {
  font-size: 0.75em;
  font-weight: 600;
  color: #999;
  padding: 4px 10px;
  margin-bottom: 2px;
}

.menu-item {
  padding: 8px 10px;
  cursor: pointer;
  font-size: 0.9em;
  color: #37352f;
  border-radius: 4px;
  transition: background 0.1s;
}

.menu-item:hover, .menu-item.is-selected {
  background-color: #f1f1ef;
}

.status-bar {
  position: fixed;
  bottom: 24px;
  right: 32px;
  font-size: 13px;
  color: #b3b3b1;
  pointer-events: none;
  transition: opacity 0.3s ease;
}
</style>