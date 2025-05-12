<!-- 
* @Description:现在： 生成之后的文件 是一个 form表单，没有使用 composition api 
* @Date:2024-12-21 20:31:57
* @Version:1.0
* @Author:will
-->
<template>
  <el-form ref="elForm" :model="formData" :rules="rules" size="medium" label-width="100px">
    <el-form-item label="单行文本" prop="field114">
      <el-col :span="24">
        <el-input v-model="formData.field114" placeholder="请输入" clearable :style="{ width: '100%' }"></el-input>
      </el-col>
    </el-form-item>
    <el-form-item label="多行文本" prop="field115">
      <el-col :span="12">
        <el-input v-model="formData.field115" type="textarea" placeholder="请输入" :autosize="{ minRows: 4, maxRows: 4 }" :style="{ width: '100%' }"></el-input>
      </el-col>
    </el-form-item>
  </el-form>
</template>
<script>
  import { ref } from 'vue'
  export default {
    setup() {
      const elForm = ref(null)
      const formData = ref({
        field114: undefined,
        field115: undefined
      })
      const rules = ref({
        field114: [
          {
            required: true,
            message: '请输入',
            trigger: 'blur'
          }
        ],
        field115: [
          {
            required: true,
            message: '请输入',
            trigger: 'blur'
          }
        ]
      })
      return {
        rules,
        formData
      }
    }
  }
</script>
<style></style>
