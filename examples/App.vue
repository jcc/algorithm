<template>
  <div class="container my-5">
    <a-star></a-star>
    <foot-bar></foot-bar>
  </div>
</template>

<script>
import AStar from './components/AStar.vue'
import FootBar from './components/Footer'
import './css/styles.css'

export default {
  components: {
    AStar,
    FootBar,
  },
}
</script>
