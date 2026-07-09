<script setup>
import { ref } from 'vue'

// 核心状态：这就是我们整篇文章的底层灵魂
const blocks = ref([
  { id: '1', type: 'h1', content: 'AI 智能笔记' },
  { id: '2', type: 'p', content: '尝试在这里输入文字，或者敲击回车...' }
])
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
      >
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

.block-item {
  margin-bottom: 5px;
}

/* 去除 input 默认丑陋的边框，让它看起来像普通的文本一样 */
.block-input {
  width: 100%;
  border: none;
  outline: none;
  background: transparent;
  resize: none;
}

/* 当我们把 input 聚焦时，给个极浅的背景色提示这是可编辑的 */
.block-input:focus {
  background-color: #f9f9fa;
  border-radius: 4px;
}

/* 针对不同类型的块，赋予不同的文字样式 */
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