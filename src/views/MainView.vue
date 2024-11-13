<template>
  <div class="main-view">
    <!-- 控制面板放置在左侧 -->
    <div class="header">风电场无人机巡检平台</div>
    <div class="control-panel">
      <el-tabs v-model="activeTab" tab-position="left">
        <!-- 地形管理 -->
        <el-tab-pane label="地形管理" name="terrain">
          <TerrainUpload @terrainLoaded="onTerrainLoaded" />
        </el-tab-pane>

        <!-- 风机管理 -->
        <el-tab-pane label="风机管理" name="turbine">
          <!-- 确保 scene 和 terrainMesh 正确传递 -->
          <WindTurbineManager
            :scene="scene"
            :terrainMesh="terrainMesh"
            :scaleFactor="scaleFactor"
          />
        </el-tab-pane>

        <!-- 其他模块保留不变 -->
        <el-tab-pane label="机库管理" name="hangar"></el-tab-pane>
        <el-tab-pane label="路径规划" name="route"></el-tab-pane>
        <el-tab-pane label="无人机巡检" name="inspection"></el-tab-pane>
      </el-tabs>
    </div>

    <!-- 场景容器放在右侧 -->
    <div class="scene-container" ref="sceneContainer"></div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import TerrainUpload from '@/components/TerrainUpload.vue'
import WindTurbineManager from '@/components/WindTurbineManager.vue'

const activeTab = ref('terrain')
const sceneContainer = ref(null)

// 场景相关变量
let scene, camera, renderer, controls
const terrainMesh = ref(null) // 定义 terrainMesh

// 初始化场景
const initScene = () => {
  scene = new THREE.Scene()
  window.THREE_SCENE = scene

  camera = new THREE.PerspectiveCamera(
    75,
    sceneContainer.value.clientWidth / sceneContainer.value.clientHeight,
    0.1,
    10000,
  )
  camera.position.set(0, 100, 200)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setPixelRatio(window.devicePixelRatio)
  renderer.setClearColor(0xf0f0f0)
  renderer.setSize(sceneContainer.value.clientWidth, sceneContainer.value.clientHeight)
  sceneContainer.value.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05

  // 灯光设置
  const ambientLight = new THREE.AmbientLight(0x404040)
  scene.add(ambientLight)
  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(1, 1, 1)
  scene.add(directionalLight)

  // 辅助工具
  const axesHelper = new THREE.AxesHelper(100)
  scene.add(axesHelper)
  const gridHelper = new THREE.GridHelper(1000, 100)
  scene.add(gridHelper)
}

// 地形加载完成时的回调函数
const scaleFactor = ref(1) // 默认的 scaleFactor

const onTerrainLoaded = (mesh, calculatedScaleFactor) => {
  terrainMesh.value = mesh
  scaleFactor.value = calculatedScaleFactor // 使用计算出的缩放比例
}

// 处理窗口大小变化
const handleResize = () => {
  if (!sceneContainer.value) return
  const width = sceneContainer.value.clientWidth
  const height = sceneContainer.value.clientHeight
  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

// 动画循环
const animate = () => {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

// 生命周期钩子
onMounted(() => {
  initScene()
  animate()
  window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  scene.clear()
  renderer.dispose()
  controls.dispose()
})
</script>

<style scoped>
.main-view {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: row; /* 横向排列，控制面板在左侧 */
}

.header {
  width: 100%;
  text-align: center;
  font-size: 24px;
  font-weight: bold;
  padding: 10px 0;
  background-color: #f5f5f5;
  color: #333;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  position: fixed;
  top: 0;
  z-index: 1000;
}

.control-panel {
  width: 300px; /* 控制面板的宽度 */
  background: #fff;
  padding: 20px;
  box-shadow: 2px 0 10px rgba(0, 0, 0, 0.1);
  overflow-y: auto;
  margin-top: 60px; /* 给出标题的高度空间 */
}

.scene-container {
  flex: 1; /* 场景容器占据剩余空间 */
  position: relative;
  margin-top: 60px; /* 给出标题的高度空间 */
}
</style>
