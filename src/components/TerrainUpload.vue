<template>
  <div class="terrain-upload">
    <h3>上传地形文件</h3>
    <input type="file" accept=".tif" @change="onFileUpload" />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import * as THREE from 'three'
import { fromArrayBuffer } from 'geotiff'

// 定义事件发射器
const emit = defineEmits(['terrainLoaded'])
const terrainMesh = ref(null)

// 计算缩放比例
const calculateScaleFactor = (terrainWidth, terrainHeight) => {
  const expectedSize = 1000 // 期望的场景尺寸（单位：米）
  return expectedSize / Math.max(terrainWidth, terrainHeight)
}

const onFileUpload = async (event) => {
  const file = event.target.files[0]
  if (file && file.type === 'image/tiff') {
    try {
      const arrayBuffer = await file.arrayBuffer()
      const tiff = await fromArrayBuffer(arrayBuffer)
      const image = await tiff.getImage()
      const rasters = await image.readRasters()

      // 获取地形图像的宽度和高度
      const width = image.getWidth()
      const height = image.getHeight()
      const data = rasters[0]

      // 下采样以提高性能
      const maxDataPoints = 4096
      const downsampleFactor = Math.ceil(Math.sqrt((width * height) / maxDataPoints))
      const downsampledData = []
      for (let y = 0; y < height; y += downsampleFactor) {
        for (let x = 0; x < width; x += downsampleFactor) {
          downsampledData.push(data[y * width + x])
        }
      }

      const minElevation = Math.min(...downsampledData)
      const maxElevation = Math.max(...downsampledData)

      // 计算缩放比例
      const scaleFactor = calculateScaleFactor(width, height)

      // 渲染地形
      renderTerrain(
        downsampledData,
        width / downsampleFactor,
        height / downsampleFactor,
        minElevation,
        maxElevation,
        scaleFactor, // 将缩放比例传递给 renderTerrain
      )
    } catch (error) {
      console.error('Error reading TIFF file:', error)
    }
  } else {
    alert('请选择一个 .tif 文件')
  }
}

// 渲染地形
const renderTerrain = (data, width, height, minElevation, maxElevation, scaleFactor) => {
  const terrainWidth = 1000
  const terrainHeight = 1000

  const geometry = new THREE.PlaneGeometry(terrainWidth, terrainHeight, width - 1, height - 1)
  geometry.rotateX(-Math.PI / 2)

  const positions = geometry.attributes.position.array
  const colors = new Float32Array(data.length * 3)

  for (let i = 0; i < data.length; i++) {
    const elevation = data[i]
    positions[i * 3 + 1] = ((elevation - minElevation) / (maxElevation - minElevation)) * 100

    const color = getElevationColor(elevation, minElevation, maxElevation)
    colors[i * 3] = color.r
    colors[i * 3 + 1] = color.g
    colors[i * 3 + 2] = color.b
  }

  geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3))
  geometry.attributes.position.needsUpdate = true
  geometry.computeVertexNormals()

  const material = new THREE.MeshLambertMaterial({ vertexColors: true, side: THREE.DoubleSide })
  const newTerrainMesh = new THREE.Mesh(geometry, material)

  if (terrainMesh.value) {
    window.THREE_SCENE.remove(terrainMesh.value)
    terrainMesh.value.geometry.dispose()
    terrainMesh.value.material.dispose()
  }

  terrainMesh.value = newTerrainMesh
  window.THREE_SCENE.add(newTerrainMesh)

  // 触发 terrainLoaded 事件，并传递缩放比例和地形网格
  emit('terrainLoaded', newTerrainMesh, scaleFactor)
}

// 高程颜色映射
const getElevationColor = (elevation, minElevation, maxElevation) => {
  const normalizedElevation = (elevation - minElevation) / (maxElevation - minElevation)
  const colorStops = [
    { threshold: 0, color: hexToRgb('#193C17') },
    { threshold: 0.2, color: hexToRgb('#2B573A') },
    { threshold: 0.4, color: hexToRgb('#527D54') },
    { threshold: 0.6, color: hexToRgb('#C4A484') },
    { threshold: 0.8, color: hexToRgb('#8B4513') },
    { threshold: 1.0, color: hexToRgb('#F5F5F5') },
  ]

  for (let i = 0; i < colorStops.length - 1; i++) {
    if (normalizedElevation <= colorStops[i + 1].threshold) {
      const t =
        (normalizedElevation - colorStops[i].threshold) /
        (colorStops[i + 1].threshold - colorStops[i].threshold)
      const color1 = colorStops[i].color
      const color2 = colorStops[i + 1].color
      return {
        r: color1.r + (color2.r - color1.r) * t,
        g: color1.g + (color2.g - color1.g) * t,
        b: color1.b + (color2.b - color1.b) * t,
      }
    }
  }
  return colorStops[colorStops.length - 1].color
}

// HEX 转 RGB
const hexToRgb = (hex) => {
  const shorthandRegex = /^#?([a-f\d])([a-f\d])([a-f\d])$/i
  hex = hex.replace(shorthandRegex, (m, r, g, b) => r + r + g + g + b + b)

  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex)
  return result
    ? {
        r: parseInt(result[1], 16) / 255,
        g: parseInt(result[2], 16) / 255,
        b: parseInt(result[3], 16) / 255,
      }
    : { r: 0, g: 0, b: 0 }
}
</script>

<style scoped>
.terrain-upload {
  margin: 10px;
}
</style>
