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

// ==========================================
// 3. 高级交互逻辑：斜杠 / 菜单
// ==========================================
const showMenu = ref(false)
const menuPosition = ref({ top: '0px', left: '0px' })
const activeBlockIndex = ref(null)

const menuOptions = [
  { type: 'h1', label: '一级标题 (H1)' },
  { type: 'h2', label: '二级标题 (H2)' },
  { type: 'p', label: '普通段落 (Text)' }
]
const selectedMenuIndex = ref(0)

const handleInput = (index, event) => {
  const currentBlock = blocks.value[index]
  if (currentBlock.content.endsWith('/')) {
    const rect = event.target.getBoundingClientRect()
    menuPosition.value = {
      // 稍微往右偏移一点，视觉上更靠近文字起点
      top: `${rect.bottom + window.scrollY + 5}px`,
      left: `${rect.left + window.scrollX + 20}px`
    }
    showMenu.value = true
    activeBlockIndex.value = index
    selectedMenuIndex.value = 0 // 每次弹出来，默认选中第一项
  } else {
    showMenu.value = false
  }
}

const turnInto = (newType) => {
  if (activeBlockIndex.value !== null) {
    const block = blocks.value[activeBlockIndex.value]
    block.type = newType
    block.content = block.content.slice(0, -1) // 删掉 /

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

        <input type="text" v-model="block.content" :class="['block-input', `block-${block.type}`]"
          :placeholder="block.type === 'h1' ? '无标题' : '输入 \'/\' 唤出菜单...'" @keydown.enter.prevent="handleEnter(index)"
          @keydown.delete="handleBackspace(index, $event)" @keydown.up="handleArrowKey('up', $event)"
          @keydown.down="handleArrowKey('down', $event)" @input="handleInput(index, $event)"
          :ref="(el) => inputRefs[index] = el">
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
  position: fixed; /* 固定在浏览器视口 */
  bottom: 24px;
  right: 32px;
  font-size: 13px;
  color: #b3b3b1; /* 极其柔和的浅灰色 */
  pointer-events: none; /* 鼠标穿透，防止挡住底部的字点不到 */
  transition: opacity 0.3s ease;
}
</style>