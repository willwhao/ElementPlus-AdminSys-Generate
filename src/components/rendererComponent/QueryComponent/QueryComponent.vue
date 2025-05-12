<!-- 
* @Description:公用查询项   用于渲染生成之后的文件的查询项
* @Date:2024-12-21 20:31:57
* @Version:1.0
* @Author:will
-->
<template>
  <ContentWrap>
    <el-form
      :class="unfold ? 'queryForm' : 'queryForm queryFormAll'"
      :model="queryParams"
      ref="queryFormRef"
      :inline="true"
      :label-width="labelWidth"
    >
      <el-form-item
        :label="item.title"
        :prop="item.attrName"
        v-for="(item, key) in ComputedProperties()"
        :key="key"
      >
        <!-- 文字展示   注：未双向绑定 -->
        <span v-if="item.inputType == 'txt'" :class="`!w-${item.width ? item.width : '220'}px`">{{
          item.dataSource
        }}</span>
        <!-- 普通输入框 -->
        <el-input
          v-if="item.inputType == 'text'"
          v-model="queryParams[item.attrName]"
          :placeholder="`请输入${item.title}`"
          :class="`!w-${item.width ? item.width : '220'}px`"
        />
        <FileUploadComponent
          ref="fileInputRef"
          v-if="item.inputType == 'file'"
          :max-size="5"
          :allowed-types="['xlsx', 'xls']"
          placeholder="请选择表格文件"
          @file-selected="handleFileSelected"
          @validation-error="handleValidationError"
        />
        <!-- 下拉系统字典项   dict-->
        <el-select
          v-if="item.inputType == 'select' && item.selectType == 'dict'"
          @change="callBack(item.event ? item.event : '无事件', queryParams[item.attrName])"
          v-model="queryParams[item.attrName]"
          :placeholder="`请输入${item.title}`"
          clearable
          :multiple="item.multiple"
          :class="`!w-${item.width ? item.width : '220'}px`"
        >
          <el-option
            v-for="dict in getIntDictOptions(DICT_TYPE[item.DICT])"
            :key="dict.value"
            :label="dict.label"
            :value="dict.value ? dict.value : 0"
          />
        </el-select>
        <!-- obj自定义下拉 车辆字典 obj vehicleDict -->
        <el-select
          v-if="item.selectType == 'obj' || item.selectType == 'vehicleDict'"
          @change="callBack(item.event ? item.event : '无事件', queryParams[item.attrName])"
          v-model="queryParams[item.attrName]"
          :placeholder="`请输入${item.title}`"
          clearable
          :multiple="item.multiple"
          :class="`!w-${item.width ? item.width : '220'}px`"
        >
          <el-option
            v-for="dict in item.dataSource"
            :key="dict.value"
            :label="dict.label"
            :value="dict.value"
          />
        </el-select>
        <!-- 时间范围选择器 -->
        <el-date-picker
          v-if="DATE_TYPES.includes(item.inputType)"
          v-model="queryParams[item.attrName]"
          :type="item.inputType"
          range-separator="至"
          start-placeholder="开始时间"
          end-placeholder="结束时间"
          :format="item.format"
          :value-format="item.format"
          :class="`!w-${item.width ? item.width : '200'}px`"
        />
        <!-- 多选组 -->
        <el-checkbox-group
          v-if="item.inputType == 'checkBoxGroup'"
          v-model="queryParams[item.attrName]"
        >
          <el-checkbox
            v-for="(dict, index) in item.dataSource"
            :key="index"
            :label="dict.label"
            :value="dict.value"
          />
        </el-checkbox-group>
        <!-- 单选组 -->
        <el-radio-group
          v-if="item.inputType == 'radioGroup'"
          v-model="queryParams[item.attrName]"
          @change="callBack(item.event, queryParams[item.attrName])"
        >
          <el-radio
            v-for="(dict, index) in item.dataSource"
            :key="index"
            :label="dict.label"
            :value="dict.value"
          />
        </el-radio-group>
        <!-- 数据范围 -->
        <div
          v-if="item.inputType == 'dataRange'"
          class="dataRange"
          :style="{ width: item.width ? item.width + 'px' : '220px' }"
        >
          <el-input
            v-model="queryParams[item.attrName]"
            :placeholder="`请输入${item.title}`"
            clearable
          />
          <div class="between">—</div>
          <el-input
            v-model="queryParams[item.attrName1]"
            :placeholder="`请输入${item.title}`"
            clearable
          />
        </div>
        <!-- 查询项按钮 -->
        <div
          class="dataRange"
          v-if="item.inputType == 'btn'"
          :style="{ width: item.width ? item.width + 'px' : 'auto' }"
        >
          <el-button
            :type="item.btnType"
            @click="callBack(item.callBackFunction, item)"
            :disabled="item.disabled"
            >{{ item.btnTitle }}</el-button
          >
        </div>
      </el-form-item>
    </el-form>
    <div class="queryFooter" v-if="queryBtn">
      <el-button @click="getList()">查询</el-button>
      <el-button @click="resetQuery">重置</el-button>
      <el-button type="text" style="margin-left: 20px" @click="flexible" v-if="expand">{{
        unfold ? '展开' : '收起'
      }}</el-button>
    </div>
  </ContentWrap>
</template>

<script setup lang="ts">
import { getIntDictOptions, DICT_TYPE } from '@/utils/dict'
import * as dictMaintain from '@/api/vehicleManagement/dictMaintain'
import FileUploadComponent from './UploadFileInput.vue'
/*
配置项properties解释：
类型：数组-对象 [{}]
{
  title: '牌照号',//输入项名称
  width: '200',//宽度  默认220px
  inputType: {
      txt：文字展示;  为文字 时无双向绑定，需要外层特殊处理
      text：文本输入框;
      checkBoxGroup:多选组;
      radioGroup:单选组;
      daterange: 时间范围;
      dataRange: 数据范围; 用于得到两个值，如1到100内的值 ，设置后需要多传一个attrName1
      select：下拉;　
          关联selectType:下拉类型
            －dict ：系统字典下拉；  传入DICT:'VEHICLE_TYPE' 内容为大写的字典项，
            －vehicleDict ：车辆系统字典下拉；  传入DICT:'VEHICLE_TYPE' 内容为大写的字典项，
            －obj  : 定制下拉；  传入如  dataSource：[{label:'男',value:'1'},{label:'女',value:'2'}]
            －multiple:开启多选
            －value:默认值
          关联DICT:字典key值
          dataSource:数据（定制下拉时用到）
      hide:隐藏查询项， 用于默认查询参数，不显示查询项， 绑定defaultValue值

  }
  attrName: 'attr1',//属性名称 ，用于表单返回的结构，或传入后端时的属性名
  dataSource: '无'//数据来源
    关联于inputType为指定属性时的处理
    checkBoxGroup：多选组
    radioGroup:单选组;
    [
      {
        value: 1,
        label: '本单位导入'
      }
    ]
  format：时间格式，关联inputType为daterange、date时的格式  //YYYY-MM-DD hh:mm:ss
},

配置项 expand 解释：
类型：Boolean
用于控制是否显示 展开 收起 按钮

*/
//时间类型
const DATE_TYPES = [
  'year',
  'years',
  'month',
  'months',
  'date',
  'dates',
  'datetime',
  'week',
  'datetimerange',
  'daterange',
  'monthrange',
  'yearrange'
]
const props = defineProps({
  properties: {
    type: Array,
    required: true
  },
  expand: {
    type: Boolean,
    default: true
  },
  resetbtnnoxhr: {
    type: Boolean,
    default: false
  },
  labelWidth: {
    type: String,
    default: '120px'
  },
  //控制查询 重置 展开收起按钮的隐藏   用于自定义查询 处理， 行内查询 处理
  queryBtn: {
    type: Boolean,
    default: true
  }
})

const queryParams = reactive({})
//查询条件展开收起
const unfold = ref(true)
const flexible = () => {
  unfold.value = !unfold.value
}

//回调方法
const callBack = (func, item) => {
  if (func == 'getList') {
    getList()
  } else {
    emit(func, item)
  }
}

//处理隐藏项
const ComputedProperties = () => {
  return props.properties.filter((item) => item.inputType !== 'hide')
}
//查询方法
const getList = async () => {
  //处理隐藏查询项
  props.properties.forEach((item) => {
    if (item.inputType == 'hide' || item.defaultValue) {
      queryParams[item.attrName] = item.defaultValue
    }
  })

  emit('getList', queryParams)
}

//重置方法
const queryFormRef = ref() // 查询的表单
const resetQuery = async () => {
  if (props.resetbtnnoxhr) {
    queryFormRef.value.resetFields()
    emit('resetQuery', queryParams)
  } else {
    props.properties.forEach((item) => {
      if (item.attrName1) {
        queryParams[item.attrName1] = undefined
      }
    })
    queryFormRef.value.resetFields()
    getList()
    emit('resetQuery', queryParams)
  }
}

//用于父组件更改子组件 的查询项的值  案列 新增变更的查询项更改 busSubsidy.vue
const updateQueryParams = (filed, value) => {
  queryParams[filed] = value
}

/* 文件上传组件 处理 */
const selectedFile = ref(null)
const fileInputRef = ref(null)
const handleFileSelected = (file) => {
  selectedFile.value = file
  const fileItem = props.properties.find((item) => item.inputType == 'file')
  fileItem.dataSource = file
  queryParams[fileItem.attrName] = file
  console.log(fileItem, '文件已选择', file)
}
const handleValidationError = (error) => {
  console.log('验证错误：', error)
}

const emit = defineEmits(['getList', 'resetQuery'])

onMounted(() => {
  //处理车辆字典

  props.properties.forEach(async (item, i) => {
    if (item.value) {
      queryParams[item.attrName] = item.value
    }
    if (item.selectType == 'vehicleDict') {
      await dictMaintain
        .getVehicleDictValue({ dataItemCode: item.DICT, pageSize: -1 })
        .then((res) => {
          item.dataSource = res.list
        })
    }
  })
})
defineExpose({
  updateQueryParams
})
</script>

<style lang="scss" scoped>
.queryForm {
  max-height: 90px;
  overflow: hidden;
  transition: 0.5s;
}
.queryFormAll {
  max-height: 1000px;
}
.queryFooter {
  padding: 5px 15px;
  text-align: right;
}

// 数据范围
.dataRange {
  display: flex;
  justify-content: space-between;

  & > div {
    width: 40%;
  }
  .between {
    width: 10% !important;
    text-align: center;
  }
}
</style>
