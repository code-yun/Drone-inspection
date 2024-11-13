<!-- src/views/MainView.vue -->
<template>
  <div class="main-view">
    <div class="scene-container" ref="sceneContainer"></div>
    <div class="control-panel">
      <el-tabs v-model="activeTab">
        <el-tab-pane label="地形管理" name="terrain">
          <!-- 后续添加地形管理组件 -->
        </el-tab-pane>
        <el-tab-pane label="风机管理" name="turbine">
          <!-- 后续添加风机管理组件 -->
        </el-tab-pane>
        <el-tab-pane label="机库管理" name="hangar">
          <!-- 后续添加机库管理组件 -->
        </el-tab-pane>
        <el-tab-pane label="路径规划" name="route">
          <!-- 后续添加路径规划组件 -->
        </el-tab-pane>
      </el-tabs>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'

const activeTab = ref('terrain')
const sceneContainer = ref(null)

// 场景相关变量
let scene, camera, renderer, controls

// 初始化场景
const initScene = () => {
  // 创建场景
  scene = new THREE.Scene()

  // 创建相机
  camera = new THREE.PerspectiveCamera(
    75,
    sceneContainer.value.clientWidth / sceneContainer.value.clientHeight,
    0.1,
    10000,
  )
  camera.position.set(0, 100, 200)

  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setPixelRatio(window.devicePixelRatio)
  renderer.setClearColor(0xf0f0f0)
  renderer.setSize(sceneContainer.value.clientWidth, sceneContainer.value.clientHeight)
  sceneContainer.value.appendChild(renderer.domElement)

  // 创建控制器
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05

  // 添加光源
  const ambientLight = new THREE.AmbientLight(0x404040)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(1, 1, 1)
  scene.add(directionalLight)

  // 添加坐标轴辅助
  const axesHelper = new THREE.AxesHelper(100)
  scene.add(axesHelper)

  // 添加网格辅助
  const gridHelper = new THREE.GridHelper(1000, 100)
  scene.add(gridHelper)
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
  // 清理Three.js资源
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
}

.scene-container {
  flex: 1;
  position: relative;
}

.control-panel {
  width: 400px;
  background: #fff;
  padding: 20px;
  box-shadow: -2px 0 10px rgba(0, 0, 0, 0.1);
  overflow-y: auto;
}
</style>
