<template>
  <flexbox
    :class="{ 'is-unread' :data.is_read == 0 }"
    class="message-cell"
    align="stretch">
    <div class="message-cell__hd">
      <i
        :class="typeObj.icon"
      />
    </div>

    <div class="message-cell__bd">
      <span>{{ leftContent }}</span>
      <span :class="[{ 'is-invalid': isInvalid }, 'click-content']" @click="enterDetail">{{ centerCotent }}</span>
      <span>{{ rightContent }}</span>
    </div>

    <div class="message-cell__ft">
      <div>{{ data.create_time | formatTime }} </div>
      <div class="handle">
        <span class="check" @click="messageReadClick">标记已读</span>
        <span class="delete" @click="messageDeleteClick">删除</span>
      </div>
    </div>
  </flexbox>
</template>

<script>
// import moment from 'moment'
export default {
  // 消息cell
  name: 'MessageCell',
  components: {

  },
  props: {
    data: Object,
    dataIndex: [String, Number]
  },
  data() {
    return {}
  },
  computed: {
    typeObj() {
      const typesObj = {
        leads: {
          icon: 'wk wk-leads',
          color: '#6995FF',
          type: 'leads'
        },
        customer: {
          icon: 'wk wk-customer',
          color: '#6995FF',
          type: 'customer'
        },
        contacts: {
          icon: 'wk wk-contacts',
          color: '#6995FF',
          type: 'contacts'
        },
        business: {
          icon: 'wk wk-business',
          color: '#6995FF',
          type: 'business'
        },
        contract: {
          icon: 'wk wk-contract',
          color: '#6995FF',
          type: 'contract'
        },
        receivables: {
          icon: 'wk wk-receivables',
          color: '#6995FF',
          type: 'receivables'
        },
        product: {
          icon: 'wk wk-product',
          color: '#6995FF',
          type: 'product'
        },
        log: {
          icon: 'wk wk-log',
          color: '#6995FF',
          type: 'log'
        },
        examine: {
          icon: 'wk wk-approve',
          color: '#6995FF',
          type: 'examine'
        },
        task: {
          icon: 'wk wk-o-task',
          color: '#6995FF',
          type: 'task'
        },
        announcement: {
          icon: 'wk wk-announcement',
          color: '#6995FF',
          type: 'announcement'
        },
        schedule: {
          icon: 'wk wk-schedule',
          color: '#6995FF',
          type: 'schedule'
        },
        invoice: {
          icon: 'wk wk-invoice',
          color: '#6995FF',
          type: 'invoice'
        }
      }

      // 1 任务 2 日志 3 oa审批 4公告 5 日程 7 客户管理
      let key = ''
      if (this.data.label && this.data.label <= 5) {
        key = ['task', 'log', 'examine', 'announcement', 'schedule'][this.data.label - 1]
      } else {
        if ([1, 2, 3, 27].includes(this.data.type)) {
          key = 'task'
        } else if ([4, 5, 34].includes(this.data.type)) {
          key = 'log'
        } else if ([6, 7, 8, 2500].includes(this.data.type)) {
          key = 'examine'
        } else if ([9].includes(this.data.type)) {
          key = 'announcement'
        } else if ([12, 13, 23, 11, 30, 33].includes(this.data.type)) {
          key = 'contract'
        } else if ([15, 16, 2700, 14].includes(this.data.type)) {
          key = 'receivables'
        } else if ([17, 1500, 21, 29, 32].includes(this.data.type)) {
          key = 'customer'
        } else if ([18, 1700].includes(this.data.type)) {
          key = 'contacts'
        } else if ([19, 1900].includes(this.data.type)) {
          key = 'leads'
        } else if ([20, 2100].includes(this.data.type)) {
          key = 'product'
        } else if ([22, 28, 31].includes(this.data.type)) {
          key = 'business'
        } else if ([25, 26, 24].includes(this.data.type)) {
          key = 'invoice'
        } else if ([10].includes(this.data.type)) {
          key = 'schedule'
        }
      }

      return typesObj[key]
    },

    leftContent() {
      return {
        1: `${this.data.user_name}将`,
        2: `${this.data.user_name}邀请您参与`,
        3: `${this.data.user_name}将`,
        4: `${this.data.user_name}回复了您的`,
        5: `${this.data.user_name}将日志`,
        6: `${this.data.user_name}提交 审批，请及时处理。`,

        7: `${this.data.user_name}拒绝您的`,
        8: `${this.data.user_name}已经审核通过您的`,
        9: `您有一个新公告`,
        10: `${this.data.user_name}邀请您参与`,

        12: `${this.data.user_name}拒绝您的`,
        13: `${this.data.user_name}已经审核通过您的`,
        14: `${this.data.user_name}提交 `,
        15: `${this.data.user_name}拒绝您的`,
        16: `${this.data.user_name}已经审核通过您的`,
        17: `${this.data.user_name}导入客户数据，${this.getImportContent(this.data)}`,
        1500: `${this.data.user_name}取消导入客户数据，已导入，${this.getImportContent(this.data)}`,
        18: `${this.data.user_name}导入联系人数据，${this.getImportContent(this.data)}`,
        1700: `${this.data.user_name}取消导入联系人数据，已导入${this.data.title}条，${this.getImportContent(this.data)}`,
        19: `${this.data.user_name}导入线索数据，${this.getImportContent(this.data)}`,
        1900: `${this.data.user_name}取消导入线索数据，已导入，${this.getImportContent(this.data)}`,
        20: `${this.data.user_name}导入产品数据，${this.getImportContent(this.data)}`,
        2100: `${this.data.user_name}取消导入产品数据，已导入，${this.getImportContent(this.data)}`,
        22: `${this.data.user_name}将您添加为商机`,
        21: `${this.data.user_name}将您添加为客户`,
        23: `${this.data.user_name}将您添加为合同`,
        2500: `${this.data.user_name}提交了`,
        11: `${this.data.user_name}提交了`,
        2700: `${this.data.user_name}提交了`,
        28: `${this.data.user_name}退出了您商机`,
        29: `${this.data.user_name}退出了您客户`,
        30: `${this.data.user_name}退出了您合同`,
        31: `${this.data.user_name}将您移出了商机`,
        32: `${this.data.user_name}将您移出了客户`,
        33: `${this.data.user_name}将您移出了合同`,
        34: `${this.data.user_name}回复了您评论的日志`,
        25: `${this.data.user_name}拒绝您的`,
        26: `${this.data.user_name}已经审核通过您的`,
        24: `${this.data.user_name}提交了`,
        27: `${this.data.user_name}项目任务导入数据，${this.getImportContent(this.data)}`
      }[this.data.type]
    },

    centerCotent() {
      // 导入提示与其他不一样
      if (this.isImportType) {
        // title 是总数 content 是错误数据 valid 错误文件是否有效 1 有效 0 失效
        const list = this.data.content.split(',') || []
        const errSize = Number(list[3] || 0)
        if (errSize > 0) {
          return this.data.valid === 0 ? '已失效' : '点击下载错误数据'
        }
        return ''
      } else {
        return `《${this.data.title}》`
      }
    },

    isInvalid() {
      if (this.isImportType) {
        // title 是总数 content 是错误数据 valid 错误文件是否有效 1 有效 0 失效
        return this.data.valid == 0
      } else {
        return false
      }
    },

    /**
     * 是导入type
     */
    isImportType() {
      return (this.data.type >= 17 && this.data.type <= 20) || this.data.type == 27
    },

    rightContent() {
      return {
        1: `任务分配给您，请及时查看`,
        2: `任务，请及时查看`,
        3: `任务标记结束`,
        4: `日志：“${this.data.content}”，请及时查看`,
        5: `发送给您，请及时查看`,
        6: `，请及时查看`,
        7: `，拒绝理由：“${this.data.content}”，请及时处理`,
        8: `，请及时查看`,
        9: `，请及时查看`,
        10: `的日程，${this.getStartTime(this.data.content)}请及时查看`,
        12: `合同审批，拒绝理由：“${this.data.content}”，请及时处理`,
        13: `合同，请及时查看`,
        14: ` 回款审批待您处理。`,
        15: `回款审批，拒绝理由：“${this.data.content}”，请及时处理`,
        16: `回款，请及时查看`,
        17: ``,
        1500: ``,
        18: ``,
        1700: ``,
        19: ``,
        1900: ``,
        20: ``,
        2100: ``,
        22: `的成员`,
        21: `的成员`,
        23: `的成员`,
        2500: `，请及时处理`,
        11: `合同审批，请及时处理`,
        2700: `回款审批，请及时处理`,
        28: `的团队`,
        29: `的团队`,
        30: `的团队`,
        31: `的团队`,
        32: `的团队`,
        33: `的团队`,
        34: `：“${this.data.content}”，请及时查看`,
        25: `发票审批，拒绝理由：“${this.data.content}”，请及时处理`,
        26: `发票，请及时查看`,
        24: `发票审批，请及时处理`,
        27: ``
      }[this.data.type]
    }
  },
  watch: {},
  mounted() {},

  beforeDestroy() {},
  methods: {
    enterDetail() {
      if (this.isInvalid) {
        return
      }

      // 是导入提醒
      if (this.isImportType) {
        this.$emit('download', this.data, this.dataIndex)
      } else {
        // 未读触发读
        if (this.data.is_read == 0) {
          this.$emit('read', this.data.message_id, this.dataIndex)
        }
        this.$emit('detail', this.typeObj.type, this.data.action_id, this.dataIndex, this.data)
      }
    },

    /**
     * 标记已读
     */
    messageReadClick() {
      this.$emit('read', this.data.message_id, this.dataIndex)
    },

    /**
     * 消息删除
     */
    messageDeleteClick() {
      this.$emit('delete', this.data.message_id, this.dataIndex)
    },

    /**
     * 日程提醒，多长时间后开始
     */
    getStartTime(content) {
      if (this.data.type != 10) {
        return
      }
      const timeObj = content || { type: 0, value: '' }

      let dataValue = ''
      if (!timeObj.type) {
        dataValue = ''
      } else {
        dataValue = '将于' + timeObj.value + ['', '分钟', '小时', '天'][timeObj.type] + '后开始, '
      }
      return dataValue
    },

    getImportContent({ title, content }) {
      const countList = [17, 1500, 18, 1700, 19, 1900, 20, 2100, 50, 27]
      if (!countList.includes(this.data.type)) {
        return
      }
      const list = content.split(',') || []

      const updateSize = Number(list[1] || '0')
      const errSize = Number(list[3] || '0')
      return `覆盖${updateSize}条，导入成功${Number(list[2] || '0')}条，导入失败${errSize}条。`
    }
  }
}
</script>

<style lang="scss" scoped>
.message-cell {
  position: relative;
  font-size: 14px;
  color: #333;
  padding: 15px 20px 15px 15px;
  line-height: 1.5;

  &__hd {
    flex-shrink: 0;
    margin-right: 15px;
    background-color: $xr-color-primary;
    text-align: center;
    width: 28px;
    height: 28px;
    line-height: 28px;
    border-radius: 14px;

    .wk {
      color: white;
      font-size: 13px;
    }
  }

  &__bd {
    flex: 1;
    white-space: pre-wrap;
    word-wrap: break-word;
    word-break: break-all;
  }

  &__ft {
    flex-shrink: 0;
    font-size: 12px;
    color: #999;
    width: 85px;
    margin-left: 35px;
    text-align: right;
    position: relative;

    .handle {
      position: absolute;
      top: 20px;
      right: 0;
      font-size: 12px;
      color: #999;
      .check,
      .delete {
        visibility: hidden;
        cursor: pointer;
      }

      .check + .delete {
        margin-left: 5px;
      }

      .check:hover {
        color: $xr-color-primary;
      }
      .delete:hover {
        color: #F56C6C;
      }
    }
  }

  &.is-unread::before {
    content: '';
    position: absolute;
    top: 20px;
    right: 8px;
    width: 6px;
    height: 6px;
    border-radius: 3px;
    background-color: #f94e4e;
  }

  &:hover {
    background-color: #f7f8fa;
    .delete {
      visibility: visible;
    }
  }

  &.is-unread:hover {
    .check {
      visibility: visible;
    }
  }
}


.click-content {
  color: $xr-color-primary;
  cursor: pointer;

  &:hover {
    text-decoration: underline;
  }
}

.is-invalid {
  color: #999;
  pointer-events: none;
  cursor: initial;
}
</style>
