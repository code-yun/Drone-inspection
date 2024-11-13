<template>
  <div ref="container"></div>
</template>

<script setup>
import { onMounted, ref, onBeforeUnmount } from 'vue'
import * as THREE from 'three'

const container = ref(null)
let scene, camera, renderer, windTurbine, animationFrame

// 创建风机模型的函数
function createWindTurbine() {
  const turbineGroup = new THREE.Group()

  // 创建塔筒
  const towerGeometry = new THREE.CylinderGeometry(0.5, 0.5, 10, 32)
  const towerMaterial = new THREE.MeshBasicMaterial({ color: 0xaaaaaa })
  const tower = new THREE.Mesh(towerGeometry, towerMaterial)
  tower.position.y = 5
  turbineGroup.add(tower)

  // 创建叶片
  const bladeGeometry = new THREE.BoxGeometry(0.1, 5, 0.3)
  const bladeMaterial = new THREE.MeshBasicMaterial({ color: 0x333333 })
  const blades = []

  for (let i = 0; i < 3; i++) {
    const blade = new THREE.Mesh(bladeGeometry, bladeMaterial)
    blade.position.y = 10
    blade.position.z = 2.5
    blade.rotation.z = (i * (2 * Math.PI)) / 3
    turbineGroup.add(blade)
    blades.push(blade)
  }

  // 添加旋转动画
  turbineGroup.animateBlades = () => {
    blades.forEach((blade) => {
      blade.rotation.z += 0.01
    })
  }

  return turbineGroup
}

// 初始化 Three.js 场景
function initThree() {
  scene = new THREE.Scene()
  camera = new THREE.PerspectiveCamera(
    75,
    container.value.clientWidth / container.value.clientHeight,
    0.1,
    1000,
  )
  camera.position.z = 20

  renderer = new THREE.WebGLRenderer()
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  container.value.appendChild(renderer.domElement)

  windTurbine = createWindTurbine()
  scene.add(windTurbine)

  animate()
}

// 动画循环
function animate() {
  animationFrame = requestAnimationFrame(animate)
  windTurbine.animateBlades()
  renderer.render(scene, camera)
}

// 生命周期管理
onMounted(() => {
  initThree()
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrame)
  renderer.dispose()
})
</script>

<style scoped>
/* 确保渲染容器的尺寸 */
div {
  width: 100%;
  height: 100%;
}
</style>
