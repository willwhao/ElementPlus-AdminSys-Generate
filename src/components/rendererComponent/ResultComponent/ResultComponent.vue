<!-- 
* @Description:公用结果项   用于渲染生成之后的文件的结果项
* @Date:2024-12-21 20:31:57
* @Version:1.0
* @Author:will
-->
<template>
  <ContentWrap>
    <!-- 操作工具栏 -->
    <el-row
      :gutter="10"
      class="mb5 toolbar"
      style="display: flex; justify-content: space-between; align-items: center"
      v-if="toolbar?.length > 0"
    >
      <el-row :gutter="10">
        <el-col :span="1.5" v-for="(item, key) in toolbar" :key="key">
          <el-dropdown @command="handleMoreClick" v-if="item.btnType == 'tableCustomCol'">
            <el-button @click="callBack(item.callBackFunction, item)" :v-hasPermi="item.hasPermi">
              {{ item.title }}
            </el-button>
            <template #dropdown>
              <div class="checkBoxComtainer">
                <el-checkbox-group
                  v-model="customColValue"
                  class="customCol"
                  @change="changeCustomCol"
                >
                  <el-checkbox
                    v-for="(dict, index) in tableConfig"
                    :key="index"
                    :label="dict.title"
                    :value="dict.prop"
                  />
                </el-checkbox-group>
              </div>
            </template>
          </el-dropdown>

          <el-button
            v-if="item.btnType == 'dialog'"
            @click="callBack(item.callBackFunction, item)"
            :v-hasPermi="item.hasPermi"
          >
            {{ item.title }}
          </el-button>

          <el-button
            v-if="item.btnType == 'Excel'"
            @click="convertExcel(item.title)"
            :v-hasPermi="item.hasPermi"
          >
            {{ item.title }}
          </el-button>
        </el-col>
      </el-row>
    </el-row>
    <!-- width: 100%; height: calc(100vh - 310px); overflow: auto -->
    <div style="">
      <!-- AutoResizer 自动调节大小 -->
      <el-auto-resizer>
        <!-- 列表 :show-overflow-tooltip="true" -->
        <el-table
          :height="height"
          :data="tableData"
          :stripe="true"
          :show-summary="summary"
          :scrollbar-always-on="true"
          :highlight-current-row="highlightCurrentRow"
          @current-change="currentChange"
          @selection-change="handleSelectionCHange"
        >
          <!-- 多选 -->
          <el-table-column
            v-if="selection"
            type="selection"
            align="left"
            width="80"
            :fixed="tableConfig[1].fixed"
          />
          <!-- 序号 -->
          <el-table-column
            v-if="index"
            type="index"
            label="序号"
            align="center"
            width="80"
            :fixed="tableConfig[1].fixed"
          />

          <template v-for="(item, key) in tableConfig" :key="key">
            <!-- 默认表头 
              :width="item.width == 'auto' ? '' : item.width ? item.width : 220"
             
            -->
            <el-table-column
              v-if="!item.noShowCol"
              :label="item.title"
              align="center"
              :render-header="(column) => renderHeadToContent(column, tableData)"
              :prop="item.prop"
              :width="item.width"
              :fixed="item.fixed"
            >
              <!-- 多级表头处理 -->
              <template v-if="item.children">
                <el-table-column
                  v-for="(it, ke) in item.children"
                  :key="ke"
                  :label="it.title"
                  align="center"
                  :render-header="(column) => renderHeadToContent(column, tableData)"
                  :prop="it.prop"
                  :width="it.width"
                >
                  <template v-if="it.children">
                    <el-table-column
                      v-for="(i, kes) in it.children"
                      :key="kes"
                      :label="i.title"
                      align="center"
                      :render-header="(column) => renderHeadToContent(column, tableData)"
                      :prop="i.prop"
                      :width="i.width"
                    />
                  </template>
                  <template #default="scope" v-if="it.type == 'input'">
                    <span v-if="scope.row['deptName'] != '合计'">
                      <el-input
                        v-model="scope.row[it.prop]"
                        @blur="callBack(it.callBackFunc, scope.row, it.prop)"
                      />
                    </span>
                    <span v-if="scope.row['deptName'] == '合计'">
                      {{ scope.row[it.prop] }}
                    </span>
                  </template>
                </el-table-column>
              </template>

              <!-- 字典内容处理 -->
              <template #default="scope" v-if="item.type == 'dict'">
                <dict-tag :type="DICT_TYPE[item.dictValue]" :value="scope.row[item.prop]" />
              </template>
              <!-- 自定义字典内容处理 -->
              <template #default="scope" v-else-if="item.type == 'staticDict'">
                <pre>{{ customStaticDict(item.dataSource, scope.row[item.prop]) }}</pre>
              </template>
              <!-- 弹窗内容处理 -->
              <template #default="scope" v-else-if="item.type == 'dialog'">
                <el-button link @click="callBack(item.callBackFunc, scope.row)">{{
                  scope.row[item.prop]
                }}</el-button>
              </template>
              <!-- 双层备注处理  车辆牌证挂失-->
              <template #default="scope" v-else-if="item.type == 'noteLink'">
                <div>
                  <div>
                    {{ scope.row[item.prop] }}
                  </div>
                  <div>
                    <a
                      href="javascript:void(0)"
                      @click="getReviewed(computedAttachment(scope.row['attachment']))"
                      >{{ computedAttachment(scope.row['attachment'])?.name }}</a
                    >
                  </div>
                </div>
              </template>
              <!-- 自定义字典 -->
              <template #default="scope" v-else-if="item.type == 'selfdict'">
                <span v-if="scope.row.returnStatus == 1">已重新分配</span>
                <span v-if="scope.row.returnStatus == 2">被驳回</span>
                <span v-if="scope.row.returnStatus == 0">待审核</span>
              </template>
              <!-- 自定义字典1 -->
              <template #default="scope" v-else-if="item.type == 'customDict2'">
                <!-- <span>{{ item.dictData[scope.row.reviewedStatus2] }}</span> -->
                <span v-if="scope.row.reviewedStatus2 == 1">确认待审核</span>
                <span v-if="scope.row.reviewedStatus2 == 2">退回待审核</span>
                <span v-if="scope.row.reviewedStatus2 == 0"></span>
              </template>
              <template #default="scope" v-else-if="item.type == 'customDict1'">
                <!-- <span>{{ item.dictData[scope.row.reviewedStatus2] }}</span> -->
                <span v-if="scope.row.reviewedStatus1 == 1">确认</span>
                <span v-if="scope.row.reviewedStatus1 == 2">退回</span>
                <span v-if="scope.row.reviewedStatus1 == 0"></span>
              </template>
              <template #default="scope" v-else-if="item.type == 'attachmentsName'">
                <span>
                  {{ scope.row[item.prop][0]?.name }}
                </span>
              </template>
              <!-- 操作新 -->
              <template #default="scope" v-else-if="item.type == 'operation'">
                <span>
                  <el-button
                    v-for="(itm, kk) in item.mains"
                    :key="kk"
                    link
                    @click="callBack(itm.callBackFunc, scope.row, itm.type)"
                    >{{ itm.name }}</el-button
                  >
                </span>
              </template>

              <!-- 操作 -->
              <template #default="scope" v-else-if="item.prop == ''">
                <span v-if="!item.showHandle || scope.row.reviewedStatus2 == 0">
                  <span v-if="!scope.row.returnStatus">
                    <el-button
                      v-for="(itm, kk) in item.mains"
                      :key="kk"
                      link
                      @click="callBack(item.callBackFunc, scope.row, itm.type)"
                      >{{ itm.name }}</el-button
                    >
                  </span>
                  <span v-if="scope.row.returnStatus && scope.row.returnStatus == 0">
                    <el-button
                      v-for="(itm, kk) in item.mains"
                      :key="kk"
                      link
                      @click="callBack(item.callBackFunc, scope.row, itm.type)"
                      >{{ itm.name }}</el-button
                    >
                  </span>
                </span>
              </template>
            </el-table-column>
          </template>
        </el-table>
        <div class="pageBox" v-if="page">
          <el-pagination
            background
            :currentPage="pageconfig.pageNo"
            :page-size="pageconfig.pageSize"
            :page-sizes="[10, 20, 30, 40]"
            layout="total,sizes, prev,pager,next,jumper"
            :total="pageconfig.total"
            @size-change="handleSizeChange"
            @current-change="handleCurrentChange"
          />
        </div>
      </el-auto-resizer>
    </div>
  </ContentWrap>
</template>

<script setup lang="ts">
import { getIntDictOptions, DICT_TYPE } from '@/utils/dict'
import { object } from 'vue-types'
import { renderHeadToContent } from '@/utils/index'
import * as bugPropertyHeatingApi from '@/api/subsidyIssueManagement/bugPropertyHeating'

/*
配置项 toolbar 解释：
类型：数组-对象 [{}]  Array
用于控制工具栏的展现   如果 没有工具栏 责不传
{
  title: '导出表格',//按钮名称
  hasPermi: [],//按钮权限
  btnType: 'Excel',//按钮类型
      Excel ：此功能使用较多，直接本组件 内处理
      dialog ：打开弹窗  此项会通过 callBack 函数返回当前 按钮数据到父组件 ，用于打开对应弹窗  关联 callBackFunction 属性
      checkBoxGroup :多选组;
      daterange : 时间范围;
      select ：下拉;
  callBackFunction： 回调函数
      当btnType为dialog时的回调函数
  btnStyle:按钮样式  对应 element 的button 的type属性  如: type="primary"
},

配置项 tableConfig 解释：
类型：Array
用于控制表格的列
[{
  title: '型  号',  //表头名
  prop: 'prop5',    //对应参数
  width: '120'      //当前列宽度
  type:'',          //默认不传
       如传入 dict 责需要配置 dicValue
       如传入 dialog 责需要配置 callBackFunc
  dicValue:'VEHICLE_MAINTENANCE_LEVEL',   字典项
  callBackFunc：'vehicleDetail'    控制弹窗  需要在父组件接收此事件
  children: [       //是否嵌套 用于控制多级表头
      {
        title: '工时总金额',
        prop: 'prop6',
        width: '120'
      }
    ]
}]

行内编辑/删除 默认回调dialogshow
{
    title: '操作',
    prop: '',
    callBackFunc: 'dialogshow',
    mains:[{
      name:'编辑',
      type:'edit',
      hasPermi:[],
    },{
      name:'删除',
      type:'del',
      hasPermi:[],
    }]
  }

配置项 tableData 解释：
类型：Array
表格数据 同element的el-table Data
*/
const props = defineProps({
  toolbar: {
    type: Array
  },
  tableData: {
    type: Array
  },
  tableConfig: {
    type: Array
  },
  //多选
  selection: {
    type: Boolean
  },
  //序号
  index: {
    type: Boolean
  },
  //合计
  summary: {
    type: Boolean
  },
  //高度
  height: {
    type: String
  },
  //高亮当前行
  highlightCurrentRow: {
    type: Boolean,
    default: false
  },
  page: {
    type: Boolean,
    default: false
  },
  pageconfig: Object
})

// const queryParams = ref({
//   pageSize: 10,
//   pageNo: 1,
//   total: 10
// })

//默认值计算处理
const customColValue = computed(() => {
  return props.tableConfig.filter((item) => !item.noShowCol).map((item) => item.prop)
})

//计算自定义字典
const customStaticDict = (obj, val) => {
  return obj.find((item) => item.value == val)?.label
}

//自定义表格列显示
const changeCustomCol = (item) => {
  emit('changeCustomCol', item)
}
const callBack = (func, item, type) => {
  console.log(func, item)
  emit(func, item, type)
}

//导出表格
const convertExcel = (val) => {
  console.log('导出表格', val)
  emit('convertExcel', val)
}

//处理双层备注处理 附件
const computedAttachment = (val) => {
  const file = JSON.parse(val)
  return file.attachment[0]
}
//处理双层备注的预览
const getReviewed = (val) => {
  window.open(`${val?.domain}${val?.path}`, '_blank')
}

const handleSizeChange = (val) => {
  emit('pagesizechange', val)
}

const handleCurrentChange = (val) => {
  emit('pagechange', val)
}

const handleSelectionCHange = (val) => {
  console.log('handleSelectionCHange', val)
  emit('handleSelectionCHange', val)
}

const currentChange = (val) => {
  console.log('选中当前行', val)
  emit('currentChange', val)
}
onMounted(() => {})

const emit = defineEmits([])
</script>

<style lang="scss" scoped>
.pageBox {
  overflow: hidden;
  padding: 15px;
  display: flex;
  justify-content: right;
}

.linkStyle {
  color: #0b79c5;
}

.checkBoxComtainer {
  max-height: 40vh;
  overflow: auto;
}
.customCol {
  display: flex;
  flex-direction: column;
  padding: 10px;
  // width: 180px;
}

// .el-button.is-link {
//   color: #0b79c5;
// }
</style>
