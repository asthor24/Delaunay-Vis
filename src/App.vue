<template>
  <DelaunayTriangulator/>
</template>

<script>
import DelaunayTriangulator from './components/DelaunayTriangulator.vue';

export default {
  name: 'App',
  components: {
    DelaunayTriangulator
}
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 10x;
}
</style>
