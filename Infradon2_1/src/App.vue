<script setup lang="ts">
// ✅ CORRECT: App.vue imports the component it wants to display.
import Session4_TheWelcome from './components/Session4_TheWelcome.vue'
</script>

<template>
  <header>
    <h1>Mon Application</h1>
  </header>

  <main>
    <Session4_TheWelcome />
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
  padding-bottom: 2rem;
}
main {
  padding: 1rem;
}
</style>
