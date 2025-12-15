<template>
  <div class="shade" @click.once="copyColor" :style="{ background: color }">
    <p class="title">{{ color }}</p>
  </div>
</template>

<script setup>
const props = defineProps({
  color: String,
})

function copyColor() {
  navigator.clipboard.writeText(props.color)
}
</script>

<style scoped>

* {
  cursor: pointer
}

.title {
    font-size: 0.9em;
    color:rgb(255, 255, 255);
    margin:-3px;
    margin-left: 0;
}

.shade {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 65px;
  height: 35px;
  border-radius: 5px;
  border: 1px solid #00000033;
}
</style>