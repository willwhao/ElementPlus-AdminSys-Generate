<template>
  <div class="upload-container">
    <el-input
      v-model="fileName"
      :placeholder="placeholder"
      readonly
      class="upload-input"
      :class="{ 'is-error': errorMessage }"
    >
      <template #append>
        <el-button @click="handleBrowse" class="browse-button">浏览</el-button>
      </template>
    </el-input>
    <input
      ref="fileInput"
      type="file"
      @change="handleFileChange"
      :accept="acceptTypes.join(',')"
      style="display: none"
    />
    <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
  </div>
</template>

<script setup>
const props = defineProps({
  maxSize: {
    type: Number,
    default: 10
  },
  allowedTypes: {
    type: Array,
    default: () => ['xlsx', 'xls', 'csv']
  },
  placeholder: {
    type: String,
    default: '请选择文件'
  }
})

const emit = defineEmits(['file-selected', 'validation-error'])

const fileName = ref('')
const fileInput = ref(null)
const errorMessage = ref('')
const selectedFile = ref(null)

const acceptTypes = computed(() => {
  return props.allowedTypes.map((type) => `${type}`)
})

const isValidFileType = (file) => {
  if (!file) return false
  const extension = file.name.split('.').pop().toLowerCase()
  return props.allowedTypes.includes(extension)
}

const isValidFileSize = (file) => {
  if (!file) return false
  const maxSizeInBytes = props.maxSize * 1024 * 1024
  return file.size <= maxSizeInBytes
}

const handleBrowse = () => {
  errorMessage.value = ''
  fileInput.value.click()
}

const handleFileChange = (event) => {
  const file = event.target.files[0]

  errorMessage.value = ''
  fileName.value = ''
  selectedFile.value = null

  if (!file) return

  //验证文件类型
  if (!isValidFileType(file)) {
    const allowedTypesText = props.allowedTypes.join(', ')
    errorMessage.value = `只能上传${allowedTypesText}格式的表格文件`
    ElMessage.error(errorMessage.value)
    emit('validation-error', { type: 'invalid-type', message: errorMessage.value })
    fileInput.value.value = '' //清空输入
    return
  }

  if (!isValidFileSize(file)) {
    errorMessage.value = `文件大小不能超过${props.maxSize}MB`
    ElMessage.error(errorMessage.value)
    emit('validation-error', { type: 'invalid-size', message: errorMessage.value })
    fileInput.value.value = ''
    return
  }

  //通过验证，更新状态
  fileName.value = file.name
  selectedFile.value = file
  emit('file-selected', file)
}
defineExpose({
  reset: () => {
    fileName.value = ''
    errorMessage.value = ''
    selectedFile.value = null
    if (fileInput.value) {
      fileInput.value.value = ''
    }
  },
  getFile: () => selectedFile.value
})
</script>
