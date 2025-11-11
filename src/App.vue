<template>
  <div id="app">
    <header class="app-header">
      <h1>Vue 3 + Leaflet 地图展示</h1>
      <p>点、线、多边形数据可视化</p>
    </header>
    <main class="app-main">
      <Leaflet />
    </main>
  </div>
</template>

<script>
import Leaflet from '@/components/Leaflet_demo';

export default {
  name: 'App',
  components: {
    Leaflet
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: #f5f7fa;
}

.app-header {
  background: linear-gradient(90deg, #2c3e50, #4a6491);
  color: white;
  padding: 20px 30px;
  text-align: center;
}

.app-header h1 {
  font-size: 2rem;
  margin-bottom: 8px;
}

.app-header p {
  opacity: 0.9;
  font-weight: 300;
}

.app-main {
  height: calc(100vh - 120px);
}
</style>