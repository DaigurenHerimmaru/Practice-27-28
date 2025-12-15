<template>
  <div class="info-container">
    <p class="title">{{ title }}</p>
    <button class="copy-btn" @click="copyText">🗒️</button>
    <div ref="slotRef" class="text">
      <slot></slot>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'

const props = defineProps({
  title: String,
  on_copy: Function
})

const slotRef = ref(null)

async function copyText() {
  await nextTick()
  if (slotRef.value) {
    const text = slotRef.value.textContent || slotRef.value.innerText
    navigator.clipboard.writeText(text.trim())
  }
}
</script>

<style scoped>
.info-container { 
    position: relative;
    padding: 10px;
}

.copy-btn {
    color: white;
    position: absolute;
    top: 0px;  
    right: 0px;
    padding: 0px;
    background: transparent;
    border: none;
    border-radius: 4px;
    opacity: 0;
    cursor: pointer;
    font-size: 16px;
}

.info-container:hover .copy-btn {
  opacity: 1;
}

.title {
    font-size: 0.9em;
    color: gray;
    margin: -3px;
    margin-left: 0;
}

.text {
    font-weight: bold;
    font-size: 1em;
    margin-top: 5px;
}
</style>