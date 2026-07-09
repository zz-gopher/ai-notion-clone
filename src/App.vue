<script setup>
import { ref, nextTick } from 'vue'

const blocks = ref([
  { id: '1', type: 'h1', content: 'AI 智能笔记' },
  { id: '2', type: 'p', content: '尝试在这里输入文字，或者敲击回车...' }
])

const inputRefs = ref([])
// 控制菜单是否显示
const showMenu = ref(false)
// 记录菜单应该弹出的位置（X 和 Y 坐标）
const menuPosition = ref({ top: '0px', left: '0px' })
// 记录当前正在操作的是哪一个块
const activeBlockIndex = ref(null)

// 监听输入框的文字变化
const handleInput = (index, event) => {
  const currentBlock = blocks.value[index]
  
  // 如果用户输入了 / 并且是目前光标所在的块
  if (currentBlock.content.endsWith('/')) {
    // 拿到当前输入框在屏幕上的位置
    const rect = event.target.getBoundingClientRect()
    
    // 把菜单定位在输入框的正下方
    menuPosition.value = {
      top: `${rect.bottom + window.scrollY + 5}px`,
      left: `${rect.left + window.scrollX}px`
    }
    showMenu.value = true
    activeBlockIndex.value = index
  } else {
    // 如果输入的不是 /，或者删除了 /，就隐藏菜单
    showMenu.value = false
  }
}

// 菜单点击操作：把当前块转换成新的类型
const turnInto = (newType) => {
  if (activeBlockIndex.value !== null) {
    const block = blocks.value[activeBlockIndex.value]
    block.type = newType
    // 去掉结尾触发菜单的那个 '/'
    block.content = block.content.slice(0, -1) 
    
    showMenu.value = false
    activeBlockIndex.value = null
  }
}

// 处理敲击回车 (新增块)
const handleEnter = async (index) => {
  const newBlock = { id: Date.now().toString(), type: 'p', content: '' }
  blocks.value.splice(index + 1, 0, newBlock)
  
  await nextTick()
  
  if (inputRefs.value[index + 1]) {
    inputRefs.value[index + 1].focus()
  }
}

// ==========================================
// 核心逻辑：处理敲击退格键 (删除空块)
// ==========================================
const handleBackspace = async (index, event) => {
  const currentBlock = blocks.value[index]
  
  // 判断条件：当前块的内容是空的，并且它不是第一行 (第一行不能删)
  if (currentBlock.content === '' && index > 0) {
    // 1. 阻止浏览器的默认退格行为，防止吃掉上一行的文字
    event.preventDefault()
    
    // 2. 将这一块从数组中彻底删除
    blocks.value.splice(index, 1)
    
    // 3. 等待 Vue 重新渲染 DOM
    await nextTick()
    
    // 4. 将光标聚焦到上一行
    if (inputRefs.value[index - 1]) {
      inputRefs.value[index - 1].focus()
    }
  }
}
</script>

<template>
  <div class="editor-container">
    <div 
      v-for="(block, index) in blocks" 
      :key="block.id" 
      class="block-item"
    >
    <input 
      type="text" 
      v-model="block.content"
      :class="['block-input', `block-${block.type}`]"
      placeholder="输入内容"
      @keydown.enter.prevent="handleEnter(index)"
      @keydown.delete="handleBackspace(index, $event)"
      
      @input="handleInput(index, $event)" 
      
      :ref="(el) => inputRefs[index] = el"
    >
    </div>
    <div 
      v-show="showMenu" 
      class="command-menu"
      :style="{ top: menuPosition.top, left: menuPosition.left }"
    >
      <div class="menu-title">转换类型</div>
      <div class="menu-item" @click="turnInto('h1')">一级标题 (H1)</div>
      <div class="menu-item" @click="turnInto('h2')">二级标题 (H2)</div>
      <div class="menu-item" @click="turnInto('p')">普通段落 (Text)</div>
    </div>
  </div>
</template>

<style scoped>
.editor-container {
  max-width: 800px;
  margin: 50px auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
}

/* === 新增：悬浮命令菜单的样式 === */
.command-menu {
  position: absolute;
  background: white;
  border: 1px solid #e0e0e0;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  border-radius: 6px;
  width: 200px;
  z-index: 100; /* 确保悬浮在最上层 */
  padding: 8px 0;
}

.menu-title {
  font-size: 0.75em;
  color: #999;
  padding: 4px 12px;
  margin-bottom: 4px;
}

.menu-item {
  padding: 8px 12px;
  cursor: pointer;
  font-size: 0.9em;
  color: #333;
}

.menu-item:hover {
  background-color: #f1f1ef;
}

/* === 新增：二级标题的样式 === */
.block-h2 {
  font-size: 1.8em;
  font-weight: bold;
  margin-top: 0.8em;
  margin-bottom: 0.3em;
}

.block-item { 
  margin-bottom: 5px; 
}

.block-input {
  width: 100%;
  border: none;
  outline: none;
  background: transparent;
  resize: none;
}

.block-input:focus {
  background-color: #f9f9fa;
  border-radius: 4px;
}

.block-h1 {
  font-size: 2.5em;
  font-weight: bold;
  margin-top: 1em;
  margin-bottom: 0.5em;
}

.block-p {
  font-size: 1.1em;
  line-height: 1.6;
  color: #37352f;
}
</style>