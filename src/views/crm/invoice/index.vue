<template>
  <div>
    <c-r-m-list-head
      :search.sync="search"
      :crm-type="crmType"
      :create-fun="createClick"
      title="发票管理"
      placeholder="请输入发票号码/客户名称/合同编号"
      main-title="新建发票"
      @on-handle="listHeadHandle"
      @on-search="crmSearch"
      @on-export="exportInfos"/>
    <div
      v-empty="!crm.invoice.index"
      xs-empty-icon="nopermission"
      xs-empty-text="暂无权限"
      class="crm-container">
      <c-r-m-table-head
        ref="crmTableHead"
        :crm-type="crmType"
        :sort-data="sortData"
        :handle-fun="handleCommand"
        @filter="handleFilter"
        @handle="handleHandle"
        @scene="handleScene"/>
      <el-table
        v-loading="loading"
        id="crm-table"
        :row-height="40"
        :data="list"
        :height="tableHeight"
        :cell-class-name="cellClassName"
        :header-cell-class-name="headerCellClassName"
        use-virtual
        class="n-table--border"
        stripe
        border
        highlight-current-row
        style="width: 100%"
        @row-click="handleRowClick"
        @sort-change="sortChange"
        @header-dragend="handleHeaderDragend"
        @selection-change="handleSelectionChange">
        <el-table-column
          show-overflow-tooltip
          type="selection"
          align="center"
          width="55"/>
        <el-table-column
          v-for="(item, index) in fieldList"
          :key="index"
          :prop="item.prop"
          :label="item.label"
          :min-width="item.width"
          sortable="custom"
          show-overflow-tooltip>
          <template slot-scope="{row,column}">
            <template v-if="item.prop == 'check_status'">
              <span :style="getStatusStyle(row.check_status)" class="status-mark"/>
              <span>{{ getCRMStatusName(row.check_status) }}</span>
            </template>
            <template v-else-if="item.prop == 'invoice_type'">
              {{ fieldFormatter(row, column, row[column.property], item) }}
            </template>
            <wk-field-view
              v-else
              :props="item"
              :form_type="item.form_type"
              :value="row[column.property]"
            >
              <template slot-scope="{ data }">
                {{ fieldFormatter(row, column, row[column.property], item) }}
              </template>
            </wk-field-view>
          </template>
        </el-table-column>
        <el-table-column
          v-if="canUpdateStatus"
          :resizable="false"
          label="操作"
          fixed="right"
          width="150">
          <template slot-scope="scope">
            <el-button
              :disabled="scope.row.invoice_status == '已开票'"
              :type="scope.row.invoice_status == '已开票' ? '' : 'primary'"
              plain
              @click.native="markReceivables(scope)">{{ scope.row.invoice_status == "已开票" ? '已开票':'标记为开票' }}</el-button>
          </template>
        </el-table-column>
        <el-table-column />
        <el-table-column
          :resizable="false"
          fixed="right"
          width="40">
          <template
            slot="header"
            slot-scope="slot">
            <field-set
              :crm-type="crmType"
              @change="setSave"/>
          </template>
        </el-table-column>
        <wk-empty
          slot="empty"
          :props="{
            buttonTitle: '新建发票',
            showButton: saveAuth
          }"
          @click="createClick"
        />
      </el-table>
      <div class="p-contianer">
        <el-pagination
          :current-page="currentPage"
          :page-sizes="pageSizes"
          :page-size.sync="pageSize"
          :total="total"
          class="p-bar"
          layout="total, sizes, prev, pager, next, jumper"
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"/>
      </div>
    </div>

    <create
      v-if="isCreate"
      @save-success="refreshList"
      @close="isCreate = false"/>
    <mark-invoice
      :visible.sync="markShow"
      :reset="isResetInvoice"
      :detail="rowDetail"
      @change="refreshList"
    />

    <!-- 相关详情页面 -->
    <c-r-m-all-detail
      :visible.sync="showDview"
      :crm-type="rowType"
      :id.sync="rowID"
      :page-list="crmType == rowType ? list : []"
      :page-index.sync="rowIndex"
      class="d-view"
      @handle="handleHandle"/>

    <transfer-handle
      :crm-type="crmType"
      :selection-list="selectionList"
      :dialog-visible.sync="transferDialogShow"
      @handle="refreshList" />
  </div>
</template>

<script>
import {
  crmInvoiceDeleteIdsAPI
} from '@/api/crm/invoice'

// import XrHeader from '@/components/XrHeader'
// import XrTableHeader from '@/components/XrTableHeader'
import Create from './Create'
import MarkInvoice from './components/MarkInvoice'
// import { XhUserCell } from '@/components/CreateCom'
import TransferHandle from '../components/SelectionHandle/TransferHandle' // 转移
import CRMAllDetail from '@/views/crm/components/CRMAllDetail'
// import FilterForm from '../components/FilterForm'
// import FilterContent from '../components/FilterForm/FilterContent'

// import CheckStatusMixin from '@/mixins/CheckStatusMixin'
// import { separator } from '@/filters/vueNumeralFilter/filters'
// import { debounce } from 'throttle-debounce'
// import { mapGetters } from 'vuex'
// import { invoiceFilterFields, invoiceHeaderFields } from './js/fields'
import TableMixin from '../mixins/Table'
import { getFormFieldShowName } from '@/components/NewCom/WkForm/utils'


export default {
  name: 'Invoice', // 发票
  components: {
    Create,
    MarkInvoice,
    TransferHandle,
    CRMAllDetail
  },
  mixins: [TableMixin],
  props: {},
  data() {
    return {
      loading: false,
      crmType: 'invoice',
      // tableHeight: document.documentElement.clientHeight - 235, // 表的高度
      // filterParams: {
      //   invoiceStatus: '',
      //   invoiceNumber: '',
      //   logisticsNumber: '',
      //   customerName: '',
      //   realInvoiceDate: '',
      //   checkStatus: '',
      //   ownerUserId: []
      // },
      // filterCustomers: {
      //   customer: [],
      //   purposecustomer: []
      // },
      // scene: '',
      // scenes: [{
      //   name: '全部发票',
      //   scene_id: ''
      // }, {
      //   name: '我负责的发票',
      //   scene_id: 1
      // }, {
      //   name: '我下属的发票',
      //   scene_id: 2
      // }],
      // showFilter: false, // 控制筛选框
      // filters: invoiceFilterFields,
      // filterObj: { form: [] }, // 筛选确定数据
      list: [],
      // fieldList: invoiceHeaderFields, // invoice_status
      // currentPage: 1,
      // pageSize: 15,
      // pageSizes: [15, 30, 60, 100],
      // total: 0,
      selectionList: [], // 勾选数据 用于全局导出
      isCreate: false,
      rowDetail: {},
      markShow: false,
      isResetInvoice: false,
      transferDialogShow: false,
      showDview: false
      // rowType: '',
      // rowID: ''
    }
  },
  computed: {
    // ...mapGetters(['crm']),
    // handles() {
    //   const temps = []
    //   if (this.crm && this.crm[this.crmType]) {
    //     if (this.crm[this.crmType].delete) {
    //       temps.push({
    //         label: '删除',
    //         command: 'delete',
    //         icon: 'delete'
    //       })
    //     }

    //     if (this.crm[this.crmType].resetinvoicestatus && this.selectionList.length == 1) {
    //       temps.push({
    //         label: '重置开票信息',
    //         command: 'reset',
    //         icon: 'reset'
    //       })
    //     }

    //     if (this.crm[this.crmType].transfer) {
    //       temps.push({
    //         label: '转移',
    //         command: 'transfer',
    //         icon: 'transfer'
    //       })
    //     }
    //   }
    //   return temps
    // },
    // 是否能操作
    canUpdateStatus() {
      return this.crm && this.crm[this.crmType] && this.crm[this.crmType].setinvoice
    },
    showfieldList() {
      return this.fieldList.filter(item => item.prop !== 'invoiceStatus')
    }
    // 是否能保存
    // canSave() {
    //   return this.crm && this.crm[this.crmType] && this.crm[this.crmType].save
    // }
  },
  watch: {},
  created() {
    // 控制table的高度
    // window.onresize = () => {
    //   this.updateTableHeight()
    // }

    // this.debouncedRefreshList = debounce(500, () => {
    //   this.refreshList()
    // })
    // this.refreshList()
  },

  beforeDestroy() {},
  methods: {
    /**
     * 刷新
     */
    // refreshList() {
    //   this.handleCurrentChange(1)
    // },

    /**
     * 更改每页展示数量
     */
    // handleSizeChange(val) {
    //   this.pageSize = val
    //   this.getList()
    // },

    /**
     * 更改当前页数
     */
    // handleCurrentChange(val) {
    //   this.currentPage = val
    //   this.getList()
    // },

    /**
     * 创建点击
     */
    createClick() {
      this.isCreate = true
    },

    /**
     * 获取列表
     */
    // getList() {
    //   const params = {
    //     page: this.currentPage,
    //     limit: this.pageSize
    //   }

    //   if (this.filterObj && this.filterObj.obj && this.filterObj.obj.length > 0) {
    //     // params.searchList = this.filterObj.obj
    //     this.filterObj.obj.forEach(element => {
    //       params[ {
    //         create_user_name: 'create_user_id',
    //         owner_user_name: 'owner_user_id',
    //         // customer_name: 'customer_id',
    //         type_id_info: 'type_id',
    //         status_id_info: 'status_id',
    //         business_name: 'business_id',
    //         contacts_name: 'contacts_id',
    //         order_user_name: 'order_user_id',
    //         category_name: 'category_id',
    //         contract_num: 'contract_id',
    //         plan_id_info: 'plan_id'
    //       }[element.type] || element.type] = element
    //       // delete params[element.type].type
    //     })
    //   }

    //   if (this.scene) {
    //     params.scene_id = this.scene
    //   }


    //   crmInvoiceIndexAPI(params)
    //     .then(res => {
    //       this.list = res.data.list
    //       this.total = res.data.dataCount
    //       this.loading = false

    //       this.$nextTick(() => {
    //         document.querySelector('.el-table__body-wrapper').scrollTop = 1
    //       })
    //     })
    //     .catch(() => {
    //       this.loading = false
    //     })
    // },

    /**
     * 格式化字段
     */
    fieldFormatter(row, column, cellValue, field) {
      if (column.property == 'invoice_type') {
        return {
          1: '增值税专用发票',
          2: '增值税普通发票',
          3: '国税通用机打发票',
          4: '地税通用机打发票',
          5: '收据'
        }[row[column.property]]
      }
      // else if (column.property == 'contractMoney' || column.property == 'invoiceMoney') {
      //   return separator(row[column.property] || 0)
      // }
      if (field) {
        return getFormFieldShowName(field.formType, row[column.property])
      }
      return row[column.property] === '' || row[column.property] === null ? '--' : row[column.property]
    },

    // // 0待审核、1审核中、2审核通过、3已拒绝 4已撤回 5未提交
    // getStatusStyle(status) {
    //   return {
    //     backgroundColor: this.getCRMStatusColor(status)
    //   }
    // },

    /**
     * 开票操作
     */
    markReceivables(scope) {
      this.rowDetail = scope.row
      this.isResetInvoice = false
      this.markShow = true
    },

    /**
     * 改变负责人筛选条件
     */
    changeUserCell(data) {
      this.filterParams.ownerUserId = data.value
      this.refreshList()
    },

    /**
     * 表头勾选
     */
    // handleSelectionChange(val) {
    //   this.selectionList = val // 勾选的行
    // },

    /**
     * 列表操作
     */
    handleCommand(command) {
      if (command == 'delete') {
        this.$confirm('确定要删除吗？', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        })
          .then(() => {
            this.loading = true
            crmInvoiceDeleteIdsAPI({ id: this.selectionList.map(item => item.invoice_id) })
              .then(res => {
                this.loading = false
                this.$message({
                  type: 'success',
                  message: '删除成功'
                })
                this.refreshList()
              })
              .catch(() => {
                this.loading = false
              })
          })
          .catch(() => {})
      } else if (command == 'transfer') {
        this.transferDialogShow = true
      } else {
        this.rowDetail = this.selectionList[0]
        this.isResetInvoice = true
        this.markShow = true
      }
    },

    /**
     * 通过回调控制class
     */
    cellClassName({ row, column, rowIndex, columnIndex }) {
      if (column.property === 'invoice_apple_number') {
        return 'can-visit--underline can-visit--bold'
      } else if (
        column.property === 'customer_name' ||
        column.property === 'contract_num'
      ) {
        return 'can-visit--underline'
      } else {
        return ''
      }
    },

    headerCellClassName({ row, column, rowIndex, columnIndex }) {
      if (column.property === 'invoice_apple_number') {
        return 'header-can-visit-backgroud'
      }
      return ''
    }

    /**
     * 详情操作
     */
    // handleHandle(data) {
    //   if (['alloc', 'get', 'transfer', 'transform', 'delete', 'put_seas', 'exit-team'].includes(data.type)) {
    //     this.showDview = false
    //   }

    //   this.getList()
    // },

    /**
     * 查看详情
     */
    // handleRowClick(row, column, event) {
    //   if (column.property === 'customer_name') {
    //     this.rowID = row.customer_id
    //     this.rowType = 'customer'
    //     this.showDview = true
    //   } else if (column.property === 'contract_number') {
    //     this.rowID = row.contract_id
    //     this.rowType = 'contract'
    //     this.showDview = true
    //   } else if (column.property === 'invoice_apple_number') {
    //     this.rowID = row.invoice_id
    //     this.rowType = 'invoice'
    //     this.showDview = true
    //   } else {
    //     this.showDview = false
    //   }
    // },

    /**
     * 创建点击
     */
    // createClick() {
    //   this.isCreate = true
    // },

    /**
     * 切换场景
     */
    // sceneChange() {
    //   this.refreshList()
    // },

    // showFilterClick() {
    //   this.showFilter = true
    // },

    // handleFilter(form) {
    //   this.filterObj = form
    //   this.showFilter = false
    //   this.refreshList()
    //   this.updateTableHeight()
    // },

    // handleDeleteField(data) {
    //   this.filterObj = data.obj
    //   this.refreshList()
    //   this.updateTableHeight()
    // },

    // /**
    //  * 更新表高
    //  */
    // updateTableHeight() {
    //   var offsetHei = document.documentElement.clientHeight
    //   var removeHeight = 0

    //   if (this.filterObj && this.filterObj.obj && this.filterObj.obj.length > 0) {
    //     removeHeight = 285
    //   } else {
    //     removeHeight = 235
    //   }
    //   this.tableHeight = offsetHei - removeHeight
    // }
  }
}
</script>

<style lang="scss" scoped>
@import '../styles/table.scss';

/* /deep/ .xr-table-header {
  border-bottom: 1px solid #e6e6e6;
  border-top: 1px solid #e6e6e6;

  .scene-select {
    width: 180px;
  }
} */

.status-mark {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: $xr-border-radius-base;
}
</style>
