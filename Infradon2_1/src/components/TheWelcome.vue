<script setup lang="ts">
import { ref } from 'vue'
const counter = ref(10)
const increment = () => {
  counter.value++
}
</script>

<template>
  <h1>BONJOUR LE MONDE</h1>
  <button @click="increment">increment</button>
  <p style="color: white; font-size: 50px">{{ counter }}</p>
</template>
