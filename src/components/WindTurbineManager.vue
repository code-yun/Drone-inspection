<template>
  <div class="wind-turbine-manager">
    <h3>风机位置管理</h3>
    <input type="file" @change="handleFileUpload" accept=".txt" />
    <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
    <el-button type="primary" @click="loadTurbines">加载风机</el-button>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue'
import * as THREE from 'three'

const props = defineProps({
  scene: THREE.Scene,
  terrainMesh: THREE.Mesh,
  scaleFactor: { type: Number, default: 1 },
})
const turbines = ref([]) // 存储风机数据
const errorMessage = ref<string | null>(null)
const rotatingBlades = [] // 用于存储旋转叶片

// 用于标记是否初始化完成
const isInitialized = ref(false)

// 当 scene 和 terrainMesh 都有效时，标记为已初始化
watch(
  [() => props.scene, () => props.terrainMesh],
  ([newScene, newTerrainMesh]) => {
    if (newScene && newTerrainMesh) {
      isInitialized.value = true
      console.log('Scene and terrain are initialized. Loading turbines if data is available.')
      loadTurbines()
    }
  },
  { immediate: true }, // 确保在组件加载时立即检查
)

// 处理文件上传
const handleFileUpload = (event: Event) => {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    const content = e.target?.result as string
    turbines.value = parseTurbineData(content)
    if (isInitialized.value) {
      loadTurbines() // 如果初始化已完成，立即加载风机
    } else {
      errorMessage.value = '场景或地形未初始化，等待初始化完成后将加载风机。'
    }
  }
  reader.readAsText(file)
}

// 解析风机位置数据
function parseTurbineData(content: string) {
  const lines = content.split('\n')
  return lines
    .map((line) => {
      const [id, lon, lat, height] = line.split('\t').map((item) => item.trim())
      if (lon && lat && height && !isNaN(+lon) && !isNaN(+lat) && !isNaN(+height)) {
        return { id, lon: parseFloat(lon), lat: parseFloat(lat), height: parseFloat(height) }
      }
      return null
    })
    .filter((data) => data !== null)
}

// 创建风机模型
function createWindTurbine() {
  const turbineGroup = new THREE.Group()
  const scaleFactor = 0.708 // 根据塔筒高度计算的缩放因子
  turbineGroup.scale.set(scaleFactor, scaleFactor, scaleFactor)

  // 创建塔筒
  const towerGeometry = new THREE.CylinderGeometry(
    2 * scaleFactor,
    3 * scaleFactor,
    120 * scaleFactor,
    32,
  )
  const towerMaterial = new THREE.MeshPhongMaterial({
    color: 0xffffff,
    shininess: 100,
  })
  const tower = new THREE.Mesh(towerGeometry, towerMaterial)
  tower.position.y = 60 * scaleFactor
  tower.castShadow = true
  tower.receiveShadow = true
  turbineGroup.add(tower)

  // 创建机舱组
  const nacelleGroup = new THREE.Group()

  // 机舱主体
  const nacelleGeometry = new THREE.CylinderGeometry(
    2.5 * scaleFactor,
    2.5 * scaleFactor,
    8 * scaleFactor,
    16,
  )
  const nacelleMaterial = new THREE.MeshPhongMaterial({
    color: 0xffffff,
    shininess: 100,
  })
  const nacelle = new THREE.Mesh(nacelleGeometry, nacelleMaterial)
  nacelle.rotation.z = Math.PI / 2
  nacelle.castShadow = true
  nacelleGroup.add(nacelle)

  // 机舱后部锥形
  const backConeGeometry = new THREE.ConeGeometry(2.5 * scaleFactor, 4 * scaleFactor, 16)
  const backCone = new THREE.Mesh(backConeGeometry, nacelleMaterial)
  backCone.rotation.z = -Math.PI / 2
  backCone.position.x = 6 * scaleFactor
  backCone.castShadow = true
  nacelleGroup.add(backCone)

  nacelleGroup.position.y = 120 * scaleFactor
  turbineGroup.add(nacelleGroup)

  // 创建轮毂组
  const hubGroup = new THREE.Group()
  hubGroup.position.set(-4 * scaleFactor, 120 * scaleFactor, 0)

  // 创建轮毂
  const hubGeometry = new THREE.CylinderGeometry(
    1.5 * scaleFactor,
    1.5 * scaleFactor,
    2 * scaleFactor,
    32,
  )
  const hubMaterial = new THREE.MeshPhongMaterial({
    color: 0xdddddd,
    shininess: 100,
  })
  const hub = new THREE.Mesh(hubGeometry, hubMaterial)
  hub.rotation.z = Math.PI / 2
  hub.castShadow = true
  hubGroup.add(hub)

  // 创建叶片
  function createBlade() {
    const shape = new THREE.Shape()

    // 定义叶片的轮廓
    shape.moveTo(0, 0)
    shape.bezierCurveTo(
      1 * scaleFactor,
      5 * scaleFactor,
      1.5 * scaleFactor,
      15 * scaleFactor,
      1 * scaleFactor,
      30 * scaleFactor,
    )
    shape.bezierCurveTo(
      0.8 * scaleFactor,
      35 * scaleFactor,
      0.3 * scaleFactor,
      38 * scaleFactor,
      0,
      40 * scaleFactor,
    )
    shape.bezierCurveTo(
      -0.3 * scaleFactor,
      38 * scaleFactor,
      -0.8 * scaleFactor,
      35 * scaleFactor,
      -1 * scaleFactor,
      30 * scaleFactor,
    )
    shape.bezierCurveTo(
      -1.5 * scaleFactor,
      15 * scaleFactor,
      -1 * scaleFactor,
      5 * scaleFactor,
      0,
      0,
    )

    const extrudeSettings = {
      steps: 1,
      depth: 0.2 * scaleFactor,
      bevelEnabled: true,
      bevelThickness: 0.1 * scaleFactor,
      bevelSize: 0.1 * scaleFactor,
      bevelSegments: 3,
    }

    const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings)
    const material = new THREE.MeshPhongMaterial({
      color: 0xffffff,
      shininess: 70,
    })

    const blade = new THREE.Mesh(geometry, material)
    blade.castShadow = true
    return blade
  }

  // 创建叶片组
  const bladeGroup = new THREE.Group()

  // 创建三个叶片
  for (let i = 0; i < 3; i++) {
    const blade = createBlade()

    // 调整叶片的初始方向
    blade.rotation.y = Math.PI / 2 // 使叶片平躺
    blade.rotation.z = Math.PI * 0.05 // 设置叶片倾角

    // 创建一个容器来旋转叶片
    const bladeContainer = new THREE.Group()
    bladeContainer.add(blade)

    // 调整叶片容器的方向
    bladeContainer.rotation.z = (i * 2 * Math.PI) / 3 // 均匀分布三个叶片，绕z轴旋转

    bladeGroup.add(bladeContainer)
  }

  // 将整个叶片组旋转90度，使其绕y轴旋转
  bladeGroup.rotation.y = Math.PI / 2
  bladeGroup.rotation.x = Math.PI / 2 // 添加这行，使叶片组绕y轴旋转

  hubGroup.add(bladeGroup)
  rotatingBlades.push(bladeGroup)

  turbineGroup.add(hubGroup)

  // 调整整个风机组的方向
  turbineGroup.rotation.y = Math.PI

  return turbineGroup
}

// 获取地形高度
function getTerrainHeight(x: number, z: number) {
  if (!props.terrainMesh) {
    console.warn('Terrain mesh not found.')
    return 0
  }

  const raycaster = new THREE.Raycaster()
  const downVector = new THREE.Vector3(0, -1, 0)
  raycaster.set(new THREE.Vector3(x, 1000, z), downVector)

  const intersects = raycaster.intersectObject(props.terrainMesh)
  if (intersects.length > 0) {
    console.log(`Intersection found at (${x}, ${z}):`, intersects[0].point.y)
    return intersects[0].point.y
  } else {
    console.warn(`No intersection found for terrain at x:${x}, z:${z}`)
    return 0
  }
}

// 加载风机到场景
function loadTurbines() {
  if (!props.scene || !props.terrainMesh) {
    errorMessage.value = '场景或地形未初始化'
    return
  }

  turbines.value.forEach(({ id, lon, lat, height }) => {
    const turbine = createWindTurbine()

    // 转换地理坐标为场景坐标
    const { x, z } = convertGeoToSceneCoords(lon, lat)

    // 使用 `props.scaleFactor` 缩放坐标
    const scaledX = x * props.scaleFactor
    const scaledZ = z * props.scaleFactor

    // 获取地形高度并调整位置
    const terrainHeight = getTerrainHeight(scaledX, scaledZ)
    const y = terrainHeight

    // 设置风机位置
    turbine.position.set(scaledX, y, scaledZ)
    props.scene.add(turbine)
  })
}

// 坐标转换函数
function convertGeoToSceneCoords(lon: number, lat: number) {
  const lonMin = 111.9476
  const lonMax = 112.0913
  const latMin = 28.2918
  const latMax = 28.4182

  // 确保与地形的宽度和高度一致
  const terrainWidth = 1000
  const terrainHeight = 1000

  const x = ((lon - lonMin) / (lonMax - lonMin)) * terrainWidth - terrainWidth / 2
  const z = ((latMax - lat) / (latMax - latMin)) * terrainHeight - terrainHeight / 2
  return { x, z }
}

// 叶片旋转动画
function animateBlades() {
  rotatingBlades.forEach((bladeGroup) => {
    bladeGroup.rotation.z += 0.05
  })
  requestAnimationFrame(animateBlades)
}

onMounted(() => {
  animateBlades()
})

onUnmounted(() => {
  rotatingBlades.length = 0
})
</script>

<style scoped>
.wind-turbine-manager {
  padding: 10px;
}

.error-message {
  color: red;
  font-size: 14px;
  margin-top: 5px;
}
</style>

<style scoped>
.wind-turbine-manager {
  padding: 10px;
}

.error-message {
  color: red;
  font-size: 14px;
  margin-top: 5px;
}
</style>
