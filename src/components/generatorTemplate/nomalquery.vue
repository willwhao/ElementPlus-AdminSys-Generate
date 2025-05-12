
<!-- 
* @Description:目标： 生成之后的文件  包含查询项和结果项， 使用了 composition api 
* @Date:2024-12-21 20:31:57
* @Version:1.0
* @Author:will
-->
<template>
  <div class="DictModel">
    <div class="leftContainer">
      <ElCard>
        <CommonTree
          @node-click="handleNodeClick"
          treeTitle="单位目录树"
          :loadTreeApi="getDeptTree"
          lazy
        />
      </ElCard>
    </div>
    <div class="centerContainer">
      <ElCard>
        <QueryComponent :properties="queryProperties" @get-list="setquery" :expand="true" />
      </ElCard>
      <ElCard style="margin-top: 10px">
        <ResultComponent
          index
          v-loading="tableLoading"
          :toolbar="toolbarConfig"
          :tableData="tableData"
          :tableConfig="tableConfig"
          @get-list="getList"
          @vehicle-detail="vehicleDetail"
          @convert-excel="convertExcel"
        />
        <Pagination
          :total="pageconfig.total"
          v-model:page="pageconfig.pageNo"
          v-model:limit="pageconfig.pageSize"
          @pagination="getList"
        />
      </ElCard>
    </div>
  </div>
  <DialogVehicleDoc ref="DialogVehicleDocRef" />
</template>

<script setup lang="ts">
import QueryComponent from '@/components/Layout/QueryComponent/QueryComponent.vue'
import ResultComponent from '@/components/Layout/ResultComponent/ResultComponent.vue'
import * as PlatemanagerApi from '@/api/vehicleManagement/platemanager'
import download from '@/utils/download'
import DialogVehicleDoc from '@/components/vehicleManagement/DialogVehicleDoc.vue'

import * as DeptApi from '@/api/system/dept'
import { getIntDictOptions, DICT_TYPE } from '@/utils/dict'
/* ————————————————————目录树———————————————————— */
const userLevel = ref(localStorage.getItem('level')) //用户级别  一级 二级  三级

//单位目录树数据
const getDeptTree = async () => {
  const res = await DeptApi.getSimpleDeptList1({
    deptSource: 1,
    level: localStorage.getItem('level')
  })
  console.log('res', res)
  return res
}
//表单配置项
const queryProperties = ref([
  {
    title: '单位名称',
    inputType: 'txt',
    attrName: 'deptName',
    dataSource: '无'
  },
  {
    title: '日期',
    inputType: 'daterange',
    attrName: 'date',
    dataSource: '无'
  },
  // {
  //   title: '开始年份',
  //   inputType: 'year',
  //   attrName: 'prop1',
  //   dataSource: '无'
  // },
  // {
  //   title: '截止年份',
  //   inputType: 'year',
  //   attrName: 'prop2',
  //   dataSource: '无'
  // },
  {
    title: '供养关系',
    inputType: 'select',
    attrName: 'supportRelation',
    selectType: 'vehicleDict',
    DICT: 'support_relation'
  },
  // {
  //   title: '季度',
  //   inputType: 'select',
  //   attrName: 'prop3',
  //   selectType: 'dict',
  //   DICT: 'QUARTER'
  // },
  {
    title: '牌照号',
    inputType: 'text',
    attrName: 'plateNo',
    dataSource: ''
  },
  // {
  //   title: '截止月份',
  //   inputType: 'text',
  //   attrName: 'prop5',
  //   dataSource: '无'
  // },

  {
    title: '单位属性',
    inputType: 'select',
    attrName: 'deptType',
    selectType: 'dict',
    DICT: 'VEHICLE_UNIT_STATS'
  }
])

const toolbarConfig = ref([
  {
    title: '导出表格',
    btnStyle: 'primary',
    hasPermi: [],
    btnType: 'Excel',
    callBackFunction: ''
  }
])

// 表格配置项 标题项
const tableConfig = ref([
  {
    title: '单位名称',
    prop: 'deptName'
  },
  {
    title: '牌照号',
    prop: 'plateNo'
  },
  {
    title: '车辆属性',
    prop: 'deptType',
    type: 'dict',
    dictValue: 'VEHICLE_UNIT_STATS'
  },
  {
    title: '厂   牌',
    prop: 'brand'
  },
  {
    title: '型  号',
    prop: 'model'
  },
  {
    title: '状态',
    prop: 'vehicleStatus'
  },
  {
    title: '登记日期',
    prop: 'recordDate'
  },
  {
    title: '加油升数',
    prop: 'fuelQty'
  },
  {
    title: '加油金额',
    prop: 'fuelAmount'
  },
  {
    title: '修理费',
    prop: 'prop10',
    children: [
      {
        title: '工时总金额',
        prop: 'prop10'
      }
    ]
  },
  {
    title: '配件费 / 材料费（元）',
    prop: 'prop11',
    children: [
      {
        title: '零配件项目金额',
        prop: 'sparePartsTotalAmount'
      },
      {
        title: '油液项目金额',
        prop: 'fluidTotalAmount'
      },
      {
        title: '轮胎项目金额',
        prop: 'tireTotalAmount'
      }
    ]
  },
  {
    title: '维修保养',
    prop: 'prop14',
    children: [
      {
        title: '合计金额',
        prop: 'totalAmount'
      }
    ]
  },
  {
    title: '保险单号',
    prop: 'policyNo'
  },
  {
    title: '保险截止期',
    prop: 'deadline'
  },
  {
    title: '保费合计',
    prop: 'premiumTotal'
  },
  {
    title: '交强险',
    prop: 'compulsoryInsurance'
  },
  {
    title: '保险赔付次数',
    prop: 'compensationCount'
  },
  {
    title: '操作',
    type: 'operation',
    prop: 'operation',
    mains: [
      {
        name: '详情',
        type: 'primary',
        callBackFunc: 'vehicleDetail',
        hasPermi: []
      }
    ]
  }
])
const treeSelect = ref({})
//点击左侧树刷新内容
const handleNodeClick = async (row: { [key: string]: any }) => {
  console.log('点击左侧树刷新内容', row)
  treeSelect.value = row
  searchquery.value.deptId = row.id
  queryProperties.value[0].dataSource = row.name
  getList()
}

const pageconfig = ref({
  pageNo: 1,
  pageSize: 10,
  total: 0
})
const searchquery = ref({
  isNewCar: 0
})
const setquery = async (form?) => {
  if (form) {
    searchquery.value = form
  }
  getList()
}
// 表格数据
const tableData = ref([])
const tableLoading = ref(false)
const getList = async () => {
  tableData.value = []
  try {
    tableLoading.value = true
    const data = await PlatemanagerApi.getVehicleItemdaiyMaintain({
      ...searchquery.value,
      ...pageconfig.value
    })
    tableLoading.value = false
    tableData.value = data.list
    pageconfig.value.total = data.total
  } finally {
  }
}

/* ——————————————————  导出表格 ——————————————————  */
const convertExcel = async () => {
  var data = await PlatemanagerApi.exportItemdaiyMaintain({
    ...searchquery.value,
    ...pageconfig.value
  })
  var fileName = sessionStorage.getItem('fileName') || '牌照管理-牌证挂失.xls'
  download.excel(data, fileName)
}

const DialogVehicleDocRef = ref()
//打开车辆档案
const vehicleDetail = (res) => {
  console.log(res)
  DialogVehicleDocRef.value.open(res)
}
onMounted(() => {
  console.log(getIntDictOptions(DICT_TYPE['VEHICLE_TEDUCE_MODE']))
})
</script>

<style lang="scss" scoped>
.DictModel {
  display: flex;
  justify-content: space-between;
  height: calc(100vh - 115px);

  //左侧区域
  .leftContainer {
    width: 20%;
    .el-card {
      height: 100%;
    }
  }
  .centerContainer {
    width: 79%;
  }
}

/* CommonDoubleTree 绑定样式 */

:deep(.el-card__body) {
  padding: 10px !important;
  height: 100%;
}
</style>
