<template>
  <div class="map-container">
    <div class="map-controls">
      <div class="control-group">
        <h3>图层控制</h3>
        <div class="control-item">
          <input type="checkbox" id="toggle-points" v-model="showPoints" @change="toggleLayer('points')">
          <label for="toggle-points">显示点数据</label>
        </div>
        <div class="control-item">
          <input type="checkbox" id="toggle-lines" v-model="showLines" @change="toggleLayer('lines')">
          <label for="toggle-lines">显示线数据</label>
        </div>
        <div class="control-item">
          <input type="checkbox" id="toggle-polygons" v-model="showPolygons" @change="toggleLayer('polygons')">
          <label for="toggle-polygons">显示多边形数据</label>
        </div>
        <button @click="toggleAllLayers" class="btn-toggle-all">
          {{ allLayersVisible ? '全部隐藏' : '全部显示' }}
        </button>
      </div>

      <div class="control-group">
        <h3>数据列表</h3>
        <div class="data-list">
          <div class="data-type">
            <h4>点数据 ({{ filteredPoints.length }})</h4>
            <div class="data-items">
              <div 
                v-for="point in filteredPoints" 
                :key="`point-${point.id}`"
                class="data-item"
                @click="focusOnFeature(point, 'point')"
              >
                <span class="item-name">{{ point.name }}</span>
                <span class="item-coords">({{ point.x }}, {{ point.y }})</span>
              </div>
            </div>
          </div>

          <div class="data-type">
            <h4>线数据 ({{ filteredLines.length }})</h4>
            <div class="data-items">
              <div 
                v-for="line in filteredLines" 
                :key="`line-${line.id}`"
                class="data-item"
                @click="focusOnFeature(line, 'line')"
              >
                <span class="item-name">{{ line.name }}</span>
                <span class="item-type">{{ line.properties.type }}</span>
              </div>
            </div>
          </div>

          <div class="data-type">
            <h4>多边形数据 ({{ filteredPolygons.length }})</h4>
            <div class="data-items">
              <div 
                v-for="polygon in filteredPolygons" 
                :key="`polygon-${polygon.id}`"
                class="data-item"
                @click="focusOnFeature(polygon, 'polygon')"
              >
                <span class="item-name">{{ polygon.name }}</span>
                <span class="item-type">{{ polygon.properties.type }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div id="map" ref="mapContainer"></div>
  </div>
</template>

<script>
import { onMounted, onUnmounted, ref, reactive, computed } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

// 修复 Leaflet 默认图标在 Webpack 下的问题
delete L.Icon.Default.prototype._getIconUrl;
L.Icon.Default.mergeOptions({
  iconRetinaUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon-2x.png',
  iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
});

export default {
  name: 'MapViewer',
  setup() {
    const mapContainer = ref(null);
    let map = null;
    
    // 响应式状态
    const showPoints = ref(true);
    const showLines = ref(true);
    const showPolygons = ref(true);
    
    // 图层引用
    const layers = reactive({
      points: L.layerGroup(),
      lines: L.layerGroup(),
      polygons: L.layerGroup()
    });
    
    // 要素引用存储
    const features = reactive({
      points: new Map(),
      lines: new Map(),
      polygons: new Map()
    });

    // 模拟数据 - 在实际应用中，这里可以通过 API 获取
    const mapData = reactive({
      points: [
        { id: 1, x: 200, y: 300, name: "监测点A", type: "point", value: 85 },
        { id: 2, x: 500, y: 500, name: "监测点B", type: "point", value: 45 },
        { id: 3, x: 800, y: 200, name: "监测点C", type: "point", value: 92 },
        { id: 4, x: 400, y: 700, name: "监测点D", type: "point", value: 23 },
        { id: 5, x: 600, y: 400, name: "监测点E", type: "point", value: 78 }
      ],
      lines: [
        { 
          id: 1, 
          name: "路线1", 
          type: "line", 
          coordinates: [[100, 100], [200, 200], [300, 150], [400, 300], [500, 250]],
          properties: { type: "主干道", length: 1200 }
        },
        { 
          id: 2, 
          name: "路线2", 
          type: "line", 
          coordinates: [[600, 100], [700, 200], [800, 150], [900, 300]],
          properties: { type: "次干道", length: 800 }
        }
      ],
      polygons: [
        {
          id: 1,
          name: "区域A",
          type: "polygon",
          coordinates: [[200, 200], [400, 200], [400, 400], [200, 400], [200, 200]],
          properties: { type: "商业区", area: 40000 }
        },
        {
          id: 2,
          name: "区域B",
          type: "polygon",
          coordinates: [[600, 300], [800, 300], [800, 500], [600, 500], [600, 300]],
          properties: { type: "住宅区", area: 40000 }
        }
      ]
    });

    // 计算属性
    const allLayersVisible = computed(() => {
      return showPoints.value && showLines.value && showPolygons.value;
    });

    const filteredPoints = computed(() => {
      return showPoints.value ? mapData.points : [];
    });

    const filteredLines = computed(() => {
      return showLines.value ? mapData.lines : [];
    });

    const filteredPolygons = computed(() => {
      return showPolygons.value ? mapData.polygons : [];
    });

    // 方法
    const initMap = () => {
      if (!mapContainer.value) return;

      // 初始化地图 - 使用 Simple 坐标系
      map = L.map(mapContainer.value, {
        crs: L.CRS.Simple,
        minZoom: -2,
        maxZoom: 4
      });

      // 定义坐标范围
      const xMin = 0, xMax = 1000;
      const yMin = 0, yMax = 1000;
      const bounds = [[yMin, xMin], [yMax, xMax]];
      
      // 设置地图视图和边界
      map.setView([500, 500], 0);
      map.setMaxBounds(bounds);
      
      // 添加网格背景
      addGrid();
      
      // 添加图层到地图
      layers.points.addTo(map);
      layers.lines.addTo(map);
      layers.polygons.addTo(map);
      
      // 加载数据
      loadPoints();
      loadLines();
      loadPolygons();
      
      // 添加比例尺
      L.control.scale({ imperial: false, maxWidth: 200 }).addTo(map);
    };

    const addGrid = () => {
      const xMin = 0, xMax = 1000;
      const yMin = 0, yMax = 1000;
      
      for (let x = 100; x < xMax; x += 100) {
        L.polyline([[yMin, x], [yMax, x]], { 
          color: '#e9ecef', 
          weight: 1,
          opacity: 0.5
        }).addTo(map);
      }
      for (let y = 100; y < yMax; y += 100) {
        L.polyline([[y, xMin], [y, xMax]], { 
          color: '#e9ecef', 
          weight: 1,
          opacity: 0.5
        }).addTo(map);
      }
    };

    const createPopupContent = (feature) => {
      let content = `
        <div class="leaflet-popup-content">
          <h4>${feature.name}</h4>
          <p>类型: ${feature.type === 'point' ? '点' : feature.type === 'line' ? '线' : '多边形'}</p>
      `;
      
      if (feature.type === 'point') {
        content += `
          <p>坐标: (${feature.x}, ${feature.y})</p>
          <p>数值: ${feature.value}</p>
        `;
      } else if (feature.type === 'line') {
        content += `
          <p>长度: ${feature.properties.length} 单位</p>
          <p>类型: ${feature.properties.type}</p>
        `;
      } else if (feature.type === 'polygon') {
        content += `
          <p>面积: ${feature.properties.area} 平方单位</p>
          <p>类型: ${feature.properties.type}</p>
        `;
      }
      
      content += `<p><small>ID: ${feature.id}</small></p></div>`;
      return content;
    };

    const loadPoints = () => {
      mapData.points.forEach(point => {
        const marker = L.circleMarker([point.y, point.x], {
          radius: 6,
          fillColor: '#e74c3c',
          color: '#c0392b',
          weight: 1.5,
          opacity: 1,
          fillOpacity: 0.8
        })
        .bindPopup(createPopupContent(point))
        .addTo(layers.points);
        
        features.points.set(point.id, marker);
      });
    };

    const loadLines = () => {
      mapData.lines.forEach(line => {
        const latLngs = line.coordinates.map(coord => [coord[1], coord[0]]);
        const polyline = L.polyline(latLngs, {
          color: '#3498db',
          weight: 3,
          opacity: 0.8
        })
        .bindPopup(createPopupContent(line))
        .addTo(layers.lines);
        
        features.lines.set(line.id, polyline);
      });
    };

    const loadPolygons = () => {
      mapData.polygons.forEach(polygon => {
        const latLngs = polygon.coordinates.map(coord => [coord[1], coord[0]]);
        const polygonLayer = L.polygon(latLngs, {
          color: '#2ecc71',
          weight: 2,
          opacity: 0.8,
          fillColor: '#2ecc71',
          fillOpacity: 0.3
        })
        .bindPopup(createPopupContent(polygon))
        .addTo(layers.polygons);
        
        features.polygons.set(polygon.id, polygonLayer);
      });
    };

    const toggleLayer = (layerType) => {
      if (layerType === 'points') {
        if (showPoints.value) {
          map.addLayer(layers.points);
        } else {
          map.removeLayer(layers.points);
        }
      } else if (layerType === 'lines') {
        if (showLines.value) {
          map.addLayer(layers.lines);
        } else {
          map.removeLayer(layers.lines);
        }
      } else if (layerType === 'polygons') {
        if (showPolygons.value) {
          map.addLayer(layers.polygons);
        } else {
          map.removeLayer(layers.polygons);
        }
      }
    };

    const toggleAllLayers = () => {
      if (allLayersVisible.value) {
        showPoints.value = false;
        showLines.value = false;
        showPolygons.value = false;
        map.removeLayer(layers.points);
        map.removeLayer(layers.lines);
        map.removeLayer(layers.polygons);
      } else {
        showPoints.value = true;
        showLines.value = true;
        showPolygons.value = true;
        map.addLayer(layers.points);
        map.addLayer(layers.lines);
        map.addLayer(layers.polygons);
      }
    };

    const focusOnFeature = (feature, type) => {
      let layer;
      
      if (type === 'point') {
        layer = features.points.get(feature.id);
        if (layer) {
          map.setView([feature.y, feature.x], 2);
          layer.openPopup();
        }
      } else if (type === 'line') {
        layer = features.lines.get(feature.id);
        if (layer) {
          const bounds = layer.getBounds();
          map.fitBounds(bounds);
          layer.openPopup();
        }
      } else if (type === 'polygon') {
        layer = features.polygons.get(feature.id);
        if (layer) {
          const bounds = layer.getBounds();
          map.fitBounds(bounds);
          layer.openPopup();
        }
      }
    };

    // 生命周期
    onMounted(() => {
      initMap();
    });

    onUnmounted(() => {
      if (map) {
        map.remove();
      }
    });

    return {
      mapContainer,
      showPoints,
      showLines,
      showPolygons,
      allLayersVisible,
      filteredPoints,
      filteredLines,
      filteredPolygons,
      toggleLayer,
      toggleAllLayers,
      focusOnFeature
    };
  }
}
</script>

<style scoped>
.map-container {
  display: flex;
  height: 100vh;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.map-controls {
  width: 300px;
  background: #f8f9fa;
  padding: 20px;
  overflow-y: auto;
  border-right: 1px solid #e0e0e0;
}

#map {
  flex: 1;
  height: 100%;
}

.control-group {
  margin-bottom: 25px;
}

.control-group h3 {
  color: #2c3e50;
  margin-bottom: 15px;
  padding-bottom: 8px;
  border-bottom: 2px solid #3498db;
}

.control-item {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.control-item input {
  margin-right: 8px;
}

.btn-toggle-all {
  width: 100%;
  padding: 8px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-top: 10px;
}

.btn-toggle-all:hover {
  background: #2980b9;
}

.data-list {
  max-height: 400px;
  overflow-y: auto;
}

.data-type {
  margin-bottom: 20px;
}

.data-type h4 {
  color: #2c3e50;
  margin-bottom: 10px;
  font-size: 0.9rem;
}

.data-items {
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  background: white;
}

.data-item {
  padding: 8px 10px;
  border-bottom: 1px solid #f0f0f0;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: background 0.2s;
}

.data-item:hover {
  background: #f0f7ff;
}

.data-item:last-child {
  border-bottom: none;
}

.item-name {
  font-weight: 500;
  color: #2c3e50;
}

.item-coords, .item-type {
  font-size: 0.8rem;
  color: #666;
}

/* Leaflet 弹窗样式调整 */
:deep(.leaflet-popup-content) {
  margin: 8px 12px;
  line-height: 1.4;
}

:deep(.leaflet-popup-content h4) {
  margin: 0 0 8px 0;
  color: #2c3e50;
}

:deep(.leaflet-popup-content p) {
  margin: 4px 0;
}

:deep(.leaflet-popup-content small) {
  color: #666;
}
</style>