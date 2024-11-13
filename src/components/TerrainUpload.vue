<template>
  <div class="terrain-upload">
    <el-upload action="#" :before-upload="handleFileUpload" :show-file-list="false" accept=".tif">
      <el-button>上传地形模型(.tif)</el-button>
    </el-upload>

    <el-upload
      action="#"
      :before-upload="handleTurbineFileUpload"
      :show-file-list="false"
      accept=".txt"
    >
      <el-button>上传风机位置信息(.txt)</el-button>
    </el-upload>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import * as THREE from 'three'
import { ElMessage } from 'element-plus'

const terrainData = ref(null)
const turbineData = ref(null)

const handleFileUpload = (file) => {
  const reader = new FileReader()

  reader.onload = (e) => {
    const fileContent = e.target.result
    terrainData.value = fileContent
    ElMessage.success('地形模型上传成功')

    // 在这里处理加载地形模型 (后面会提到如何加载.tif 文件)
  }

  reader.readAsArrayBuffer(file)
  return false
}

const handleTurbineFileUpload = (file) => {
  const reader = new FileReader()

  reader.onload = (e) => {
    const fileContent = e.target.result
    turbineData.value = fileContent
    ElMessage.success('风机位置信息上传成功')

    // 在这里处理风机位置文件 (后面会提到如何解析.txt 文件)
  }

  reader.readAsText(file)
  return false
}
</script>

<style scoped>
.terrain-upload {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
</style>
