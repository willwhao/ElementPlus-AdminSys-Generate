<!-- 主页面 -->
<script lang="ts" setup>
  import { reactive, ref, watch, nextTick, onMounted } from 'vue' // 引入Vue的核心API
  import { getIdGlobal, saveIdGlobal } from '../../utils/db' // 引入获取和保存全局ID的方法

  import RightPanel from './components/RightPanel.vue' // 引入右侧面板组件

  let idGlobal = getIdGlobal() // 初始化全局ID

  // 生成器
  import FormBuilder from '../../components/formBuilder/formBuilder.vue' // 引入表单生成器组件

  // 工具类
  import { saveAs } from 'file-saver' // 引入文件保存工具
  import { deepClone, beautifierConf } from '../../utils' // 引入深拷贝和美化配置工具
  import { debounce } from 'throttle-debounce' // 引入防抖函数

  // 一些生成 js css template的一些工具方法
  import { makeUpHtml, vueTemplate, vueScript, cssStyle } from '../../components/generator/html' // 引入生成HTML、模板、脚本和样式的方法
  import { makeUpJs } from '../../components/generator/makeJs' // 引入生成JS的方法
  import { makeUpCss } from '../../components/generator/makeCss' // 引入生成CSS的方法
  import loadBeautifier from '../../utils/loadBeautifier' // 引入加载美化工具的方法

  // 供拉拽生成的组件
  import { inputComponents, selectComponents, layoutComponents, formConf } from '../../components/generator/formConfig' // 引入输入型、选择型组件和表单配置

  // 默认值
  // import darwDefault from "../../components/generator/darwDefault" // 默认值导入（已注释）

  const leftComponents = [
    // 左侧组件列表
    {
      title: '输入型组件', // 输入型组件标题
      list: inputComponents // 输入型组件列表
    },
    {
      title: '选择型组件', // 选择型组件标题
      list: selectComponents // 选择型组件列表
    },
    {
      title: '布局型组件', // 布局型组件标题
      list: layoutComponents // 布局型组件列表
    }
  ]

  const drawingList: any[] = reactive([]) // 响应式的绘图列表

  const activeId = ref('') // 当前激活的ID

  const activeData = ref([]) // 当前激活的数据

  const onEnd = (obj: any) => {
    // 拖拽结束时的回调
    if (obj.to !== obj.form) {
      console.log(true) // 如果目标和源不同，输出true
    }
  }

  const cloneComponent = (data: any) => {
    // 克隆组件
    activeData.value = data // 设置当前激活数据
    const copyData = deepClone(data) // 深拷贝数据
    createIdAndKey(copyData) // 创建ID和Key
    return copyData // 返回拷贝的数据
  }

  const activeFormItem = (currentItem: any) => {
    // 激活表单项
    activeData.value = currentItem // 设置当前激活数据
    console.log(currentItem) // 输出当前项
    activeId.value = currentItem.__config__.formId // 设置当前激活ID
    console.log(activeId.value) // 输出当前激活ID
    console.log(currentItem) // 再次输出当前项
  }

  const drawingItemCopy = (item: any, list: any) => {
    // 复制绘图项
    let clone = deepClone(item) // 深拷贝项
    clone = createIdAndKey(clone) // 创建ID和Key
    list.push(clone) // 将克隆项添加到列表
    activeFormItem(clone) // 激活克隆项
  }

  const saveIdGlobalDebounce = debounce(340, saveIdGlobal) // 防抖保存全局ID

  watch(
    () => activeId.value, // 监听activeId的变化
    val => {
      // 当activeId变化时的回调
      console.log(val) // 输出变化的值
      saveIdGlobalDebounce(val) // 防抖保存全局ID
    },
    { deep: true } // 深度监听
  )

  /**
   * 创建formId 并强制更新组件
   */
  const createIdAndKey = (item: any) => {
    // 创建ID和Key
    const config = item.__config__ // 获取配置
    config.formId = ++idGlobal // 增加全局ID并赋值给formId
    config.renderKey = `${config.formId}${+new Date()}` // 改变renderKey后可以实现强制更新组件
    if (config.layout === 'colFormItem') {
      item.__vModel__ = `field${idGlobal}` // 设置vModel
    } else if (config.layout === 'rowFormItem') {
      config.componentName = `row${idGlobal}` // 设置组件名称
      !Array.isArray(config.children) && (config.children = []) // 确保children为数组
      delete config.label // rowFormItem无需配置label属性
    }
    if (Array.isArray(config.children)) {
      config.children = config.children.map((childItem: any) => createIdAndKey(childItem)) // 递归创建子项的ID和Key
    }
    return item // 返回项
  }

  /**
   * 删除预览的某一个拖拽组件
   */
  const drawingItemDelete = (index: any, list: any[]) => {
    // 删除绘图项
    list.splice(index, 1) // 从列表中删除项
    nextTick(() => {
      const len = drawingList.length // 获取绘图列表长度
      if (len) {
        activeFormItem(drawingList[len - 1]) // 激活最后一个项
      }
    })
  }

  const dialogVisible = ref(false) // 对话框可见性
  const showFileName = ref(false) // 是否显示文件名
  const operationType = ref<string>('') // 操作类型
  const previewContent = ref('') // 预览内容

  // 导出vue文件
  const download = () => {
    // 下载方法
    dialogVisible.value = true // 显示对话框
    showFileName.value = true // 显示文件名
    operationType.value = 'download' // 设置操作类型为下载
    const codeStr = generateCode() // 生成代码字符串
    const blob = new Blob([codeStr], { type: 'text/plain;charset=utf-8' }) // 创建Blob对象
    saveAs(blob, new Date().getTime() + '.vue') // 保存文件
  }

  const generateConf = ref<any>({
    type: 'file',
  }) // 生成配置

  const formData = ref({}) // 表单数据

  const formConfig = ref(formConf) // 表单配置

  const AssembleFormData = () => {
    // 组装表单数据
    formData.value = {
      queryFields: deepClone(drawingList), // 深拷贝绘图列表
      fields: deepClone(drawingList), // 深拷贝绘图列表
      ...formConfig.value // 合并表单配置
    }
  }

  onMounted(() => {
    // 组件挂载时加载美化工具
    loadBeautifier((btf: any) => {
      beautifier = btf // 设置美化工具
    })
  })

  let beautifier: any // 美化工具

  // 生成代码
  const generateCode = () => {
    // 生成代码方法
    const { type } = generateConf // 获取生成配置中的类型
    AssembleFormData() // 组装表单数据
    const script = vueScript(makeUpJs(formData.value as any, 'file')) // 生成脚本
    const html = vueTemplate(makeUpHtml(formData.value, 'file')) // 生成HTML
    const css = cssStyle(makeUpCss(formData.value)) // 生成CSS
    return beautifier.html(html + script + css, beautifierConf.html) // 返回美化后的代码
  }

  const run = () => {
    // 运行方法
    // 这里可以添加运行表单的逻辑
    console.log('运行表单') // 输出运行表单
  }

  const showJson = () => {
    // 查看JSON方法
    const jsonStr = JSON.stringify(formData.value, null, 2) // 将表单数据转换为JSON字符串
    console.log('表单JSON:', jsonStr) // 输出JSON字符串
    dialogVisible.value = true // 显示对话框
    showFileName.value = false // 不显示文件名
    operationType.value = 'showJson' // 设置操作类型为显示JSON
    previewContent.value = jsonStr // 设置预览内容为JSON字符串
  }

  const showVue = () => {
    // 查看Vue文件方法
    const vueStr = generateCode() // 生成Vue代码字符串
    console.log('Vue文件:', vueStr) // 输出Vue代码
    dialogVisible.value = true // 显示对话框
    showFileName.value = false // 不显示文件名
    operationType.value = 'showVue' // 设置操作类型为显示Vue文件
    previewContent.value = vueStr // 设置预览内容为Vue代码
  }

  const copy = () => {
    // 复制代码方法
    const vueStr = generateCode() // 生成Vue代码字符串
    navigator.clipboard
      .writeText(vueStr) // 将代码写入剪贴板
      .then(() => {
        console.log('代码已复制到剪贴板') // 输出复制成功信息
      })
      .catch(err => {
        console.error('复制失败:', err) // 输出复制失败信息
      })
  }

  const empty = () => {
    // 清空方法
    drawingList.splice(0, drawingList.length) // 清空绘图列表
    console.log('表单已清空') // 输出清空信息
  }
</script>

<template>
  <div class="container">
    <!-- 左侧控件拖动 -->
    <div class="left-board">
      <div class="logo-wrapper">
        <div class="logo"></div>
        <!-- Logo区域 -->
      </div>
      <el-scrollbar class="left-scrollbar">
        <div class="components-list">
          <div v-for="(item, listIndex) in leftComponents" :key="listIndex">
            <div class="components-title">{{ item.title }}</div>
            <!-- 组件标题 -->
            <Draggable item-key="listIndex" class="components-draggable" :list="item.list" :group="{ name: 'componentsGroup', pull: 'clone', put: false }" :clone="cloneComponent" draggable=".components-item" :sort="false" @end="onEnd">
              <div v-for="(element, index) in item.list" :key="index" class="components-item">
                <div class="components-body">{{ element.__config__.label }}</div>
                <!-- 组件标签 -->
              </div>
            </Draggable>
          </div>
        </div>
        <div></div>
        <!-- 空白占位 -->
      </el-scrollbar>
    </div>
    <!-- 中间表单设计区域 -->
    <div class="center-board">
      <div class="action-bar">
        <el-button icon="el-icon-download" type="text" @click="download">导出vue文件</el-button>
        <!-- 导出按钮 -->
        <el-button icon="el-icon-video-play" type="text" @click="run">运行</el-button>
        <!-- 运行按钮 -->
        <el-button icon="el-icon-view" type="text" @click="showJson">查看json</el-button>
        <!-- 查看JSON按钮 -->
        <el-button icon="el-icon-view" type="text" @click="showVue">查看vue文件</el-button>
        <!-- 查看Vue文件按钮 -->
        <el-button class="copy-btn-main" icon="el-icon-document-copy" type="text" @click="copy">复制代码</el-button>
        <!-- 复制代码按钮 -->
        <el-button class="delete-btn" icon="el-icon-delete" type="text" @click="empty">清空</el-button>
        <!-- 清空按钮 -->
      </div>
      <el-scrollbar class="center-scrollbar">
        <!-- {{ drawingList }} -->
        <el-form :label-position="formConfig.labelPosition" :size="(formConfig as any).size" :gutter="formConfig.gutter" :disabled="formConfig.disabled" :label-width="formConfig.labelWidth + 'px'">
          <Draggable class="drawing-board" :list="drawingList" :animation="340" group="componentsGroup" item-key="index">
              <!-- 渲染器 -->
              <FormBuilder :options="drawingList" :formId="formConfig" :activeId="activeId" @currentItem="activeFormItem" />
          </Draggable>
          <div v-if="drawingList.length == 0" class="empty-info">从左侧拖入或点选组件进行表单设计</div>
          <!-- 空白提示 -->
        </el-form>
      </el-scrollbar>
    </div>
    <!-- 左侧设置编辑区域 -->
    <RightPanel :active-data="activeData" :form-conf="formConfig" :show-field="!!drawingList.length"></RightPanel>

    <!-- 弹窗用于预览 JSON 和 Vue 文件 -->
    <el-dialog v-model="dialogVisible" width="80%">
      <span v-if="operationType === 'showJson'">JSON 预览</span>
      <!-- JSON预览标题 -->
      <span v-if="operationType === 'showVue'">Vue 文件预览</span>
      <!-- Vue文件预览标题 -->
      <pre>{{ previewContent }}</pre>
      <!-- 预览内容 -->
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">关闭</el-button>
        <!-- 关闭按钮 -->
      </span>
    </el-dialog>
  </div>
</template>

<style lang="scss">
  @import '../../assets/style/home.scss'; // 引入全局样式
</style>
