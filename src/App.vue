<script setup>
import { ref, nextTick, computed, watch } from 'vue'

// ==========================================
// 1. 核心状态数据
// ==========================================
// 独立的页面大标题
const pageTitle = ref('AI 智能笔记')

// 积木数组（初始化只有一行空的正文）
const blocks = ref([
  { id: '1', type: 'p', content: '' }
])

// 动态计算总字数：标题长度 + 所有积木块内容的长度
const wordCount = computed(() => {
  let count = pageTitle.value.length
  blocks.value.forEach(block => {
    count += block.content.length
  })
  return count
})

// 记录最后编辑时间
const lastEditedTime = ref('刚刚')

// 侦听器魔法：只要 pageTitle 或 blocks 发生任何微小的改变，就立刻更新时间
watch([pageTitle, blocks], () => {
  const now = new Date()
  const hours = now.getHours().toString().padStart(2, '0')
  const minutes = now.getMinutes().toString().padStart(2, '0')
  lastEditedTime.value = `${hours}:${minutes}`
}, { deep: true }) // deep: true 意思是深度监听数组里每一行的变化

// 存放所有真实 input 元素的“花名册”
const inputRefs = ref([])

// ==========================================
// 2. 基础交互逻辑：回车与退格
// ==========================================
// 处理敲击回车
const handleEnter = async (index) => {
  // === 新增：如果菜单显示着，回车就是确认选择 ===
  if (showMenu.value) {
    const selectedOption = menuOptions[selectedMenuIndex.value]
    turnInto(selectedOption.type)
    return // 结束执行，不要再往下建新行了
  }

  // 下面是原有的新增行逻辑
  const newBlock = { id: Date.now().toString(), type: 'p', content: '' }
  blocks.value.splice(index + 1, 0, newBlock)
  await nextTick()
  if (inputRefs.value[index + 1]) {
    inputRefs.value[index + 1].focus()
  }
}

// 处理敲击退格键 (删除空块)
const handleBackspace = async (index, event) => {
  const currentBlock = blocks.value[index]

  // 判断条件：内容为空，且不是第一行
  if (currentBlock.content === '' && index > 0) {
    event.preventDefault() // 阻止默认吃字行为
    blocks.value.splice(index, 1) // 拔掉这一行

    await nextTick()

    if (inputRefs.value[index - 1]) {
      inputRefs.value[index - 1].focus() // 光标上移
    }
  }
}

const triggerAI = async (index) => {
  // 1. 把触发菜单的那个 "/" 删掉
  blocks.value[index].content = blocks.value[index].content.slice(0, -1)
  
  // 2. 拿到用户当前写的字，作为给 AI 的提示词
  const prompt = blocks.value[index].content
  
  // 3. 在下方立刻新建一个空的文本块，用来装 AI 吐出来的字
  const aiBlockId = Date.now().toString()
  const aiBlock = { id: aiBlockId, type: 'p', content: '' }
  blocks.value.splice(index + 1, 0, aiBlock)
  
  try {
    // 4. 呼叫你的 Java 后端 (注意端口是 8080)
    const response = await fetch('http://localhost:8080/api/ai/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ prompt: prompt })
    })

    // 5. 开启硬核的流式解析 (像接水管一样接住打字效果)
    const reader = response.body.getReader()
    const decoder = new TextDecoder('utf-8')

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      const chunk = decoder.decode(value)
      const lines = chunk.split('\n')
      
      for (const line of lines) {
        // 💡 修复 1：只要以 'data:' 开头统统拦截，不管有没有空格
        if (line.startsWith('data:')) {
          // 💡 修复 2：安全截取掉前 5 个字符，然后用 trim() 洗掉多余的空格
          const dataStr = line.substring(5).trim()
          
          if (dataStr === '[DONE]') continue
          
          try {
            const data = JSON.parse(dataStr)
            const textContent = data.choices[0].delta.content || ''
            
            const targetBlock = blocks.value.find(b => b.id === aiBlockId)
            if (targetBlock) {
              targetBlock.content += textContent
              
              // 让高度自适应（直接利用 Vue 响应式更新后的 DOM 重新计算）
              nextTick(() => {
                // 注意：Vue 编译后 DOM 里没有 v-model，所以我们换成通用选择器
                const textareas = document.querySelectorAll('.block-input')
                textareas.forEach(el => {
                  el.style.height = 'auto'
                  el.style.height = el.scrollHeight + 'px'
                })
              })
            }
          } catch (e) {
             // 忽略 JSON 解析报错
          }
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
  // === 新增：自动撑开高度的魔法 ===
  const textarea = event.target
  // 先把高度重置为 auto，让它缩小回真实文字的高度
  textarea.style.height = 'auto'
  // 然后把高度设置为它内部“卷去”的真实高度 (scrollHeight)
  textarea.style.height = textarea.scrollHeight + 'px'
  // ====================================

  // 下面是原有的菜单逻辑，保持不变
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
      // 触发 AI 逻辑
      triggerAI(activeBlockIndex.value)
    } else {
      // 原本的格式转换逻辑
      const block = blocks.value[activeBlockIndex.value]
      block.type = newType
      block.content = block.content.slice(0, -1) 
    }
    showMenu.value = false
    activeBlockIndex.value = null
  }
}

const handleArrowKey = (direction, event) => {
  // 如果菜单没出来，不管它，让浏览器正常移动光标
  if (!showMenu.value) return

  // 如果菜单出来了，拦截默认行为（防止光标乱跑）
  event.preventDefault()

  if (direction === 'up') {
    // 向上选，如果到顶了就循环到底部
    selectedMenuIndex.value = (selectedMenuIndex.value - 1 + menuOptions.length) % menuOptions.length
  } else if (direction === 'down') {
    // 向下选，如果到底了就循环到顶部
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

        <textarea v-model="block.content" :class="['block-input', `block-${block.type}`]"
          :placeholder="block.type === 'h1' ? '无标题' : '输入 \'/\' 唤出菜单...'" rows="1"
          @keydown.enter.prevent="handleEnter(index)" @keydown.delete="handleBackspace(index, $event)"
          @keydown.up="handleArrowKey('up', $event)" @keydown.down="handleArrowKey('down', $event)"
          @input="handleInput(index, $event)" :ref="(el) => inputRefs[index] = el"></textarea>
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
/* ==========================================
   基础排版
   ========================================== */
.editor-container {
  max-width: 700px;
  margin: 40px auto;
  padding: 0 40px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
  color: #37352f;
}

/* 独立大标题样式 */
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

/* ==========================================
   积木块与动画
   ========================================== */
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

/* ==========================================
   左侧悬浮抓手
   ========================================== */
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

/* ==========================================
   输入框细节
   ========================================== */
.block-input {
  flex: 1;
  border: none;
  outline: none;
  background: transparent;
  padding: 4px 2px;
  width: 100%;
  resize: none; /* 隐藏右下角的拖拽三角 */
  overflow: hidden; /* 隐藏由于计算延迟可能闪现的滚动条 */
  font-family: inherit; /* 强制继承我们设置的高级字体 */
}

.block-input::placeholder {
  color: #e0e0e0;
}

/* 聚焦时的左侧极细灰线 */
.block-input:focus {
  border-left: 2px solid #ebeced;
  padding-left: 10px;
  margin-left: -12px;
}

.block-input::selection {
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

/* ==========================================
   Vue 过渡动画魔法
   ========================================== */
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
  /* 保持离开时的宽度，防止布局跳动 */
}

/* ==========================================
   悬浮命令菜单
   ========================================== */
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

.menu-item:hover {
  background-color: #f1f1ef;
}

.menu-item.is-selected {
  background-color: #f1f1ef;
}

.status-bar {
  position: fixed;
  /* 固定在浏览器视口 */
  bottom: 24px;
  right: 32px;
  font-size: 13px;
  color: #b3b3b1;
  /* 极其柔和的浅灰色 */
  pointer-events: none;
  /* 鼠标穿透，防止挡住底部的字点不到 */
  transition: opacity 0.3s ease;
}
</style>