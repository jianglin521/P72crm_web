<template>
  <el-dialog
    :visible="visible"
    :append-to-body="true"
    :close-on-click-modal="false"
    title="放入公海"
    width="450px"
    @close="handleCancel">
    <div class="handle-box">
      <flexbox class="handle-item">
        <div class="handle-item-name">公海：</div>
        <el-select v-model="selectId" >
          <el-option
            v-for="item in list"
            :key="item.pool_id"
            :label="item.pool_name"
            :value="item.pool_id"/>
        </el-select>
      </flexbox>
    </div>
    <span
      slot="footer"
      class="dialog-footer">
      <el-button @click.native="handleCancel">取消</el-button>
      <el-button
        v-debounce="handleConfirm"
        type="primary">保存</el-button>
    </span>
  </el-dialog>
</template>

<script>
import {
  crmCustomerPutInPoolAPI,
  crmCustomerPoolNameListAPI
} from '@/api/crm/customer'

import { Loading } from 'element-ui'

export default {
  /**
   * 客户放入公海
   */
  name: 'PutPoolHandle',
  components: {
  },
  mixins: [],
  props: {
    visible: {
      type: Boolean,
      required: true,
      default: false
    },
    id: [String, Number],
    // 操作数据
    selectionList: {
      type: Array,
      default: () => {
        return []
      }
    }
  },
  data() {
    return {
      selectId: '',
      list: []
    }
  },
  computed: {},
  watch: {
    list: {
      handler() {
        this.selectId = this.list && this.list.length > 0 ? this.list[0].pool_id : ''
      },
      immediate: true
    },
    visible(val) {
      if (val && this.list.length === 0) {
        this.getList()
      }
    }
  },
  mounted() {
  },
  methods: {
    /**
     * 获取数据源
     */
    getList() {
      const loading = Loading.service({
        target: document.querySelector(`.el-dialog[aria-label="放入公海"]`)
      })
      this.list = [
        // { 'pool_id': 1, 'pool_name': '系统默认公海', 'adminUserId': null, 'memberUserId': null, 'memberDeptId': null, 'status': null, 'preOwnerSetting': null, 'preOwnerSettingDay': null, 'receiveSetting': null, 'receiveNum': null, 'remindSetting': null, 'remindDay': null, 'putInRule': null, 'createUserId': null, 'createTime': null, 'companyId': null }
      ]
      crmCustomerPoolNameListAPI()
        .then(res => {
          this.list = res.data || []
          loading && loading.close()
        })
        .catch(() => {
          loading && loading.close()
        })
    },

    /**
     * 取消选择
     */
    handleCancel() {
      this.$emit('update:visible', false)
    },

    /**
     * 确定
     */
    handleConfirm() {
      if (this.selectId) {
        const loading = Loading.service({
          target: document.querySelector(`.el-dialog[aria-label="放入公海"]`)
        })
        crmCustomerPutInPoolAPI({
          customer_id: this.selectionList.map(item => item.customer_id),
          pool_id: this.selectId
        })
          .then(res => {
            this.$message({
              type: 'success',
              message: '操作成功'
            })
            loading.close()
            this.$emit('handle', { type: 'put_seas' })
            this.handleCancel()
          })
          .catch(() => {
            loading.close()
          })
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.handle-box {
  color: #333;
  font-size: 12px;
}
.handle-item {
  padding-bottom: 15px;
  .handle-item-name {
    flex-shrink: 0;
    width: 60px;
  }
  .el-select {
    flex: 1;
  }
}
.handle-bar {
  position: relative;
  margin-top: 10px;
  height: 30px;
  button {
    float: right;
    margin-right: 10px;
  }
}
</style>
