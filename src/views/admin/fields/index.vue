<template>
  <flexbox
    v-loading="loading"
    align="flex-start"
    justify="flex-start"
    class="fields-index body">
    <flexbox-item class="body-left">
      <div class="body-left_title">字段库</div>
      <ul>
        <draggable
          :list="fieldLibList"
          :options="dragLeftConfig"
          :clone="dragLeftMove"
          class="lib-wrapper"
          @end="dragLeftEnd">
          <div
            v-for="item in fieldLibList"
            :key="item.id"
            class="lib-item"
            @click="handleLibFieldClick(item)">
            <i
              :class="item.icon"
              class="lib-item-icon" />
            <span>{{ item.name }}</span>
        </div></draggable>
      </ul>
    </flexbox-item>

    <div class="body-content">
      <flexbox
        align="flex-start"
        justify="flex-start"
        direction="column"
        class="body-content-warp">
        <el-header>
          <div class="title">编辑{{ title }}字段</div>
          <div>
            <el-button
              v-debounce="handleSave"
              type="primary">保存</el-button>
            <el-button @click="handleCancel">返回</el-button>
          </div>
        </el-header>
        <flexbox-item style="margin-left: 0" class="body-content-main">
          <el-main>
            <draggable
              :list="fieldArr"
              :options="dragListConfig"
              class="field-list"
              @end="dragListEnd">
              <flexbox
                v-for="(childArr, fatherIndex) in fieldArr"
                :key="fatherIndex"
                align="flex-start"
                justify="flex-start"
                class="field-row">
                <component
                  v-for="(field, childIndex) in childArr"
                  ref="fieldItem"
                  :key="childIndex"
                  :is="field | typeToComponentName"
                  :field="field"
                  :field-arr="fieldArr"
                  :point="[fatherIndex, childIndex]"
                  :active-point="selectedPoint"
                  @action="handleAction"
                  @child-drag-add="handleChildDragAdd"
                  @click="handleSelect([fatherIndex, childIndex])" />
              </flexbox>
            </draggable>
          </el-main>
        </flexbox-item>
      </flexbox>
    </div>

    <flexbox-item style="margin-left: 0" class="body-right">
      <setting-field
        v-if="selectedField"
        :field="selectedField"
        :point="selectedPoint"
        :field-arr="fieldArr"
        :can-transform="canTransform"
        :transform-data="transformData"
        @child-edit="handleChildEdit"
        @update-width="handleUpdateFieldWidth" />
    </flexbox-item>
  </flexbox>
</template>

<script>
import {
  customFieldHandleAPI,
  oaFieldHandleAPI,
  customFieldListAPI,
  oaExamineFieldListAPI
} from '@/api/admin/crm'
import { filedGetFieldAPI } from '@/api/crm/common'
import { isEmpty } from '@/utils/types'

import {
  FieldInput,
  FieldTextarea,
  FieldSelect,
  FieldCheckbox,
  FieldFile,
  FieldBoolean,
  FieldPercent,
  FieldPosition,
  FieldLocation,
  FieldDetailTable,
  FieldWritingSign,
  FieldDateInterval,
  FieldDescText
} from './components/FieldItem'
import SettingField from './components/SettingField'

import draggable from 'vuedraggable'

import Field from './field'
import FieldTypeLib from './fieldTypeLib'
import { objDeepCopy } from '@/utils/index'
import { typeToComponent } from './utils'
import { regexIsCRMMobile, regexIsCRMEmail } from '@/utils'

export default {
  name: 'FieldsIndex',
  components: {
    FieldInput,
    FieldTextarea,
    FieldSelect,
    FieldCheckbox,
    FieldFile,
    FieldBoolean,
    FieldPercent,
    FieldPosition,
    FieldLocation,
    FieldDetailTable,
    FieldWritingSign,
    FieldDateInterval,
    FieldDescText,
    SettingField,
    draggable
  },
  filters: {
    /** 根据type 找到组件 */
    typeToComponentName(item) {
      return typeToComponent(item)
    }
  },
  data() {
    return {
      moduleType: '', // 所属模块
      loading: false,

      fieldLibList: [], // 字段库
      dragLeftConfig: {
        group: {
          pull: 'clone',
          put: false,
          name: 'libList'
        },
        forceFallback: true,
        sort: false
      },
      movedField: null, // 被拖拽的字段库字段

      dragListConfig: {
        delay: 100,
        group: {
          name: 'list',
          put: ['libList'],
          pull: true
        },
        forceFallback: true,
        fallbackClass: 'draggingStyle'
      },
      fieldArr: [],
      rejectHandle: true, // 请求未获取前不能操作

      isChildDragAdd: false, // 是否为子组件内部拖拽添加

      selectedPoint: [null, null],
      selectedField: null,
      // 转移匹配字段源
      transformData: null
    }
  },
  computed: {
    // 能转移
    canTransform() {
      const rowNum = this.selectedPoint[0]
      const columnNum = this.selectedPoint[1]
      if (!isEmpty(rowNum) && !isEmpty(columnNum)) {
        const currentField = this.fieldArr[rowNum][columnNum]
        if (currentField.form_type === 'detail_table') return false
      }
      return this.moduleType === 'crm_leads'
    },
    title() {
      return {
        crm_leads: '线索',
        crm_customer: '客户',
        crm_contacts: '联系人',
        crm_business: '商机',
        crm_contract: '合同',
        crm_product: '产品',
        crm_receivables: '回款',
        crm_visit: '客户回访',
        crm_marketing: '市场活动'
      }[this.moduleType] || ''
    }
  },
  created() {
    this.moduleType = this.$route.params.type || ''
    this.initCom()

    // 配置转移字段
    if (this.canTransform) {
      this.getTransformField()
    }
  },
  methods: {
    /**
     * 初始化
     */
    initCom() {
      if (this.moduleType === 'crm_marketing') {
        this.fieldLibList = FieldTypeLib.filter(item => {
          return [
            'user',
            'structure',
            'file'
          ].includes(item.form_type)
        })
      } else {
        this.fieldLibList = FieldTypeLib
      }
      this.getFieldList()
    },

    /**
     * 获取字段列表
     */
    getFieldList() {
      this.loading = true
      let request = null
      const params = {}
      const config = this.$route.params

      if (this.moduleType === 'oa_examine') {
        request = oaExamineFieldListAPI
        // params.categoryId = config.id
        params.types_id = config.id
        params.types = 'oa_examine'
      } else {
        request = customFieldListAPI
      }

      if (config.label) {
        params.types = config.type
      }

      request(params)
        .then(res => {
          this.fieldArr = res.data || []

          if (res.data.length > 0) {
            this.handleSelect([0, 0])
          }
          this.rejectHandle = false
          this.loading = false
        })
        .catch(() => {
          this.loading = false
        })
    },

    /**
     * 点击左侧字段库进行添加
     * @param field {Object}
     */
    handleLibFieldClick(field) {
      console.log('lib click')
      this.movedField = field
      this.dragLeftEnd()
    },

    /**
     * 左侧字段库拖拽
     * @param field {Object}
     */
    dragLeftMove(field) {
      this.movedField = field
    },

    /**
     * 左侧字段库拖拽结束
     */
    dragLeftEnd(evt) {
      console.log('drag left end', evt)
      if (this.rejectHandle) return
      const newField = new Field({
        name: this.movedField.name,
        form_type: this.movedField.form_type
      })
      newField.style_percent = 100
      newField.operating = 255
      if (this.$route.params.label !== 'none') {
        newField.label = this.$route.params.label
      }
      if (this.movedField.form_type === 'desc_text') {
        newField.name = ''
      }
      if (this.movedField.form_type === 'detail_table') {
        // 11101000
        newField.operating = 232
        newField.fieldExtendList = []
        newField.default_value = null
        newField.remark = `添加${newField.name}`
      }
      delete newField.field_id

      if (this.isChildDragAdd) {
        // 如果是子组件内部拖拽添加
        if ([
          'detail_table',
          'desc_text',
          'handwriting_sign'
        ].includes(newField.form_type)) {
          this.$message.error('此字段内部不能添加该类型的字段')
          return
        }
        this.childDragAddEnd(newField, evt)
        return
      }

      let rowNum = null
      if (
        evt &&
        evt.pullMode === 'clone' &&
        !isEmpty(evt.newIndex)
      ) {
        rowNum = evt.newIndex
      } else {
        rowNum = this.selectedPoint[0] + 1
      }
      this.fieldArr.splice(rowNum, 0, [newField])
      this.handleSelect([rowNum, 0])
    },

    /**
     * 子组件内部(明细表格)拖拽添加
     */
    handleChildDragAdd(point, evt) {
      // console.log('child drag add')
      this.selectedPoint = point
      this.isChildDragAdd = true
    },

    /**
     * 子组件内部(明细表格)追加表格字段
     */
    childDragAddEnd(newField, evt) {
      newField.style_percent = 50
      // 10101011
      newField.operating = 171
      const findRes = FieldTypeLib.find(o => newField.form_type === o.form_type)
      if (findRes) newField.type = findRes.type

      // const rowNum = evt.newIndex
      const rowNum = this.selectedPoint[0]
      const columnNum = this.selectedPoint[1]
      const currentField = this.fieldArr[rowNum][columnNum]
      if (isEmpty(currentField.fieldExtendList)) {
        currentField.fieldExtendList = []
      }
      newField.fieldName = this.generateFieldName(currentField.fieldExtendList)
      currentField.fieldExtendList.push(newField)
      console.log('currentField: ', currentField)
      this.$set(this.fieldArr, rowNum, this.fieldArr[rowNum])
      this.handleSelect(this.selectedPoint, newField)
      this.isChildDragAdd = false
    },

    /**
     * 拖动 list
     */
    dragListEnd(evt) {
      console.log('drag list')
      // this.selectedPoint[0] = evt.newIndex
      this.selectedPoint.splice(0, 1, evt.newIndex)
    },

    /**
     * 字段操作
     * @param action {String} 动作
     * @param point {Array} 字段的坐标
     */
    handleAction(action, point) {
      switch (action) {
        case 'top':
          this.handleActionMoveTop(point)
          break
        case 'bottom':
          this.handleActionMoveBottom(point)
          break
        case 'left':
          this.handleActionExchange(point, -1)
          break
        case 'right':
          this.handleActionExchange(point, 1)
          break
        case 'copy':
          this.handleActionCopy(point)
          break
        case 'delete':
          this.handleDelete(point)
      }
    },

    /**
     * 上移字段
     * @param point {Array} 字段的坐标
     */
    handleActionMoveTop(point) {
      const row = this.fieldArr[point[0] - 1]
      if (!row || row.length === 4) return
      const field = this.fieldArr[point[0]][point[1]]
      const topField = row[0]
      if (field.form_type === 'detail_table' || topField.form_type === 'detail_table') {
        [this.fieldArr[point[0] - 1], this.fieldArr[point[0]]] = [this.fieldArr[point[0]], this.fieldArr[point[0] - 1]]
        this.handleSelect([point[0] - 1, 0])
      } else {
        // 给新行追加字段
        row.push(objDeepCopy(field))
        let config = this.getWidth(row.length)
        row.forEach(o => {
          o.style_percent = config.style_percent
        })
        this.$set(this.fieldArr, point[0] - 1, row)

        // 把字段从原来的行中删除
        const oldRow = this.fieldArr[point[0]]
        oldRow.splice(point[1], 1)
        if (oldRow.length === 0) {
          this.fieldArr.splice(point[0], 1)
        } else {
          config = this.getWidth(oldRow.length)
          oldRow.forEach(o => {
            o.style_percent = config.style_percent
          })
          this.$set(this.fieldArr, point[0], oldRow)
        }
        this.handleSelect([point[0] - 1, row.length - 1])
      }
    },

    /**
     * 下移字段
     * @param point {Array} 字段的坐标
     */
    handleActionMoveBottom(point) {
      const field = this.fieldArr[point[0]][point[1]]
      const bottomField = this.fieldArr[point[0] + 1][0]
      const row = this.fieldArr[point[0]]

      if (field.form_type === 'detail_table' || bottomField.form_type === 'detail_table' || row.length === 1) {
        [this.fieldArr[point[0] + 1], this.fieldArr[point[0]]] = [this.fieldArr[point[0]], this.fieldArr[point[0] + 1]]
        this.handleSelect([point[0] + 1, 0])
      } else {
        field.style_percent = 100
        // 把字段放到新行
        this.fieldArr.splice(point[0] + 1, 0, [field])
        // 把字段从原来的行删除
        this.fieldArr[point[0]].splice(point[1], 1)
        // 修改原来行的字段占比
        const config = this.getWidth(row.length)
        row.forEach(o => {
          o.style_percent = config.style_percent
        })
        this.$set(this.fieldArr, point[0], row)
        this.handleSelect([point[0] + 1, 0])
      }
    },

    /**
     * 左右移动交换位置
     * @param point {Array} 字段的坐标
     * @param step {Number} 1 向右移动 -1 向左移动
     */
    handleActionExchange(point, step) {
      const row = this.fieldArr[point[0]]
      const field = this.fieldArr[point[0]][point[1]]
      row.splice(point[1], 1)
      row.splice(point[1] + step, 0, field)
      this.handleSelect([point[0], point[1] + step])
    },

    /**
     * 拷贝字段
     * @param point {Array} 字段的坐标
     */
    handleActionCopy(point) {
      const field = this.fieldArr[point[0]][point[1]]
      const copyField = objDeepCopy(field)
      delete copyField.field_id
      delete copyField.fieldName
      delete copyField.field
      delete copyField.relevant
      copyField.field_type = 0
      copyField.operating = 255
      if (copyField.form_type === 'desc_text') {
        copyField.name = ''
      }
      this.fieldArr.splice(point[0] + 1, 0, [copyField])
      this.handleSelect([point[0] + 1, point[1]])
    },

    /**
     * 修改字段占比
     */
    handleUpdateFieldWidth() {
      console.log('update width')
      const row = this.fieldArr[this.selectedPoint[0]]

      // 本行占比大于100% 溢出到下一行
      const arr = []
      let child = [] // 行
      let totalWidth = 0
      for (let i = 0; i < row.length; i++) {
        const item = row[i]
        totalWidth += item.style_percent
        if (totalWidth < 100) {
          // 长度小于 100%
          child.push(item)
        } else if (totalWidth > 100) {
          // 长度大于 100%
          arr.push(objDeepCopy(child))
          child = []
          child.push(item)
          totalWidth = item.style_percent
        } else {
          // 长度等于 100%
          child.push(item)
          arr.push(objDeepCopy(child))
          child = []
          totalWidth = 0
        }
      }
      if (child.length > 0) {
        arr.push(child)
      }

      // 如果从一行变成多行
      if (arr.length > 1) {
        let rowNum = this.selectedPoint[0] // 行坐标
        let columnNum = this.selectedPoint[1] // 列坐标
        this.fieldArr.splice(rowNum, 1, ...arr)
        let step = 0
        for (let i = 0; i < arr.length; i++) {
          step += arr[i].length
          if (step >= columnNum + 1) {
            rowNum += i
            columnNum = columnNum - step + arr[i].length
            break
          }
        }
        this.handleSelect([rowNum, columnNum])
      }
    },

    getWidth(length) {
      if (length === 1) return { style_percent: 100 }
      if (length === 2) return { style_percent: 50 }
      if (length > 2) return { style_percent: 25 }
    },

    /**
     * 删除字段
     * @param point {Array} 字段的坐标
     */
    handleDelete(point) {
      this.$confirm('确定删除该自定义字段吗?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.selectedPoint = [null, null]
          this.selectedField = null

          this.fieldArr[point[0]].splice([point[1]], 1)
          // 如果当前行已经没有元素则删除
          if (this.fieldArr[point[0]].length === 0) {
            this.fieldArr.splice(point[0], 1)
          }
        })
        .catch(() => {})
    },

    /**
     * 字段列表点击选择
     * @param point {Array} 字段的坐标
     * @param field {Object} 字段
     */
    handleSelect(point, field = null) {
      console.log('index click: ', point)
      this.selectedPoint = point
      this.selectedField = field || this.fieldArr[point[0]][point[1]]
    },

    /**
     * 编辑(明细表格)内部字段
     * @param field {Object} 为空时表明为 clickoutside
     */
    handleChildEdit(field = null) {
      if (field) {
        this.selectedField = field
      } else {
        this.handleSelect(this.selectedPoint)
      }
    },

    /**
     * 生成(明细表格)内部字段fieldName
     * @param list {Array<Object>} 全部的内部字段，防止fieldName重复
     */
    generateFieldName(list) {
      const arr = list.map(o => o.fieldName)
      const generateStr = function(len) {
        const lib = 'abcdefghijklmnopqrstuvwxyz'
        let str = ''
        for (let i = 0; i < len; i++) {
          const index = Math.ceil(Math.random() * 25)
          str += lib[index]
        }
        const res = 'field_' + str
        if (arr.includes(res)) {
          return generateStr(len)
        } else {
          return res
        }
      }
      return generateStr(6)
    },

    /**
     * 保存
     */
    handleSave() {
      if (this.rejectHandle) return

      const arr = []
      this.loading = true
      // 追加坐标
      objDeepCopy(this.fieldArr).forEach((father, fatherIndex) => {
        father.forEach((child, childIndex) => {
          arr.push({
            ...child,
            form_position: `${fatherIndex},${childIndex}`
          })
        })
      })

      const limitFields = 'select|update|union|and|or|delete|insert|trancate|char|substr|ascii|declare|exec|count|master|into|drop|execute'.split('|')
      const names = []
      for (let i = 0; i < arr.length; i++) {
        const item = arr[i]
        const position = item.form_position.split(',')
        const positionStr = `第${Number(position[0]) + 1}行第${Number(position[1]) + 1}列`
        item.name = (item.name || '').trim()
        if (item.form_type !== 'desc_text') {
          if (!item.name) {
            this.$message.error(positionStr + '自定义字段，标识名不能为空')
            this.loading = false
            return
          }
          if (limitFields.includes(item.name)) {
            this.$message.error(positionStr + '自定义字段标识名与系统字段重复，请使用其他字段！')
            this.loading = false
            return
          }
          if (names.includes(item.name)) {
            this.$message.error(positionStr + '自定义字段标识名重复')
            this.loading = false
            return
          }

          if (item.form_type === 'detail_table') {
            // 明细表格
            if (isEmpty(item.fieldExtendList)) {
              this.$message.error(positionStr + '明细字段不能为空')
              this.loading = false
              return
            }
            for (let j = 0; j < item.fieldExtendList.length; j++) {
              const o = item.fieldExtendList[j]
              delete o.companyId
              delete o.id
              if (isEmpty(o.default_value)) {
                o.default_value = null
              }
              o.name = (o.name || '').trim()
              if (!o.name) {
                this.$message.error(positionStr + '明细字段，标识名不能为空')
                this.loading = false
                return
              }
              if (limitFields.includes(o.name)) {
                this.$message.error(positionStr + '明细字段标识名与系统字段重复，请使用其他字段！')
                this.loading = false
                return
              }
            }
            const _arr = item.fieldExtendList.map(o => o.name)
            if (_arr.length !== Array.from(new Set(_arr)).length) {
              this.$message.error(positionStr + '明细字段标识名重复')
              this.loading = false
              return
            }
          }
          names.push(item.name)
        } else {
          // 描述文字
          if (!isEmpty(item.default_value) && item.default_value.length > 2000) {
            this.$message.error(positionStr + '描述文字类型字段最多设置2000字')
            this.loading = false
            return
          }
        }
        if (!item.type) {
          const findRes = FieldTypeLib.find(o => o.form_type === item.form_type)
          if (findRes) item.type = findRes.type
        }

        if (item.hasOwnProperty('optionsData')) {
          delete item.optionsData
        }

        if (item.form_type === 'mobile' && item.default_value) {
        // 校验手机号
          if (!regexIsCRMMobile(item.default_value)) {
            this.$message.error('输入的手机格式有误')
            this.loading = false
            return
          }
        }
        if (item.form_type === 'email' && item.default_value) {
        // 校验邮箱
          if (!regexIsCRMEmail(item.default_value)) {
            this.$message.error('输入的邮箱格式有误')

            this.loading = false
            return
          }
        }
        if (item.form_type === 'number' && item.default_value) {
          // 校验数字类型
          const num = Number(item.default_value) // 去0
          if (isNaN(num)) {
            this.$message.error('数字输入错误')
            this.loading = false
            return
          }
          item.default_value = String(num)
          const arr = String(num).split('.')

          const len = String(num)
            .replace('.', '')
            .replace('-', '')
            .length
          const maxlength = item.form_type === 'percent' ? 10 : 15
          if (len > maxlength) {
            this.$message.error(`最多支持${maxlength}位数字（包含小数位）`)

            this.loading = false
            return
          }

          const min = isEmpty(item.minNumRestrict) ? -Infinity : Number(item.minNumRestrict || -Infinity)
          const max = isEmpty(item.maxNumRestrict) ? Infinity : Number(item.maxNumRestrict || Infinity)
          if (num < min) {
            this.$message.error('默认值不能小于最小值')

            this.loading = false
            return
          }
          if (num > max) {
            this.$message.error('默认值不能大于最大值')

            this.loading = false
            return
          }

          // null 不支持小数  0 不限制小数位
          if (isEmpty(item.precisions)) {
            this.loading = false
            return
          }
          if (item.precisions === 0) return
          if (arr.length > 1 && arr[1].length > Number(item.precisions)) {
            this.$message.error(`默认值的小数位不能大于${item.precisions}`)

            this.loading = false
            return
          }
        }
      }

      // 请求参数
      const params = {
        data: arr
      }
      const routerParams = this.$route.params
      if (routerParams.label !== 'none') {
        // params.label = routerParams.label
        params.types = routerParams.type
      }
      if (this.moduleType === 'oa_examine') {
        // params.categoryId = routerParams.id
        params.types_id = routerParams.id
      }
      if (this.moduleType === 'crm_marketing') {
        params.formId = routerParams.id
      }
      console.log('save, ', params)
      // this.loading = false
      // return

      const request = {
        oa_examine: oaFieldHandleAPI
      }[this.moduleType] || customFieldHandleAPI

      request(params)
        .then(() => {
          this.$message({
            type: 'success',
            message: '操作成功'
          })
          this.loading = false
          this.getFieldList()
        })
        .catch(() => {
          this.loading = false
        })
    },

    /**
     * 获取添加字段
     */
    getTransformField() {
      filedGetFieldAPI({
        types: 'crm_customer',
        module: 'crm',
        action: 'save',
        controller: 'customer',
        format: 1 // 一维数组
      })
        .then(res => {
          const data = {
            text: [],
            textarea: [],
            select: [],
            checkbox: [],
            number: [],
            floatnumber: [],
            mobile: [],
            email: [],
            date: [],
            datetime: [],
            user: [],
            structure: [],
            boolean_value: [],
            percent: [],
            position: [],
            location: [],
            handwriting_sign: [],
            date_interval: []
          }

          for (let index = 0; index < res.data.length; index++) {
            const element = res.data[index]
            const array = data[element.form_type]
            if (array) {
              array.push({
                label: element.name,
                value: element.field_id
              })
            }
          }

          this.transformData = data
        })
        .catch(() => {})
    },

    /**
     * 返回
     */
    handleCancel() {
      this.$router.go(-1)
    }
  }
}
</script>

<style scoped lang="scss">
@import '@/styles/mixin.scss';

.fields-index {
  &.body {
    position: relative;
    width: 100%;
    height: calc(100% + 15px);
    user-select: none;
    overflow: hidden;

    .body-left {
      width: 265px;
      max-width: 265px;
      height: 100%;
      overflow-y: auto;
      background-color: white;
      .body-left_title {
        .wk-icon-fields {
          margin-right: 10px;
        }
        font-weight: bold;
        font-size: 14px;
        margin: 20px 20px 15px;
      }

      .lib-wrapper {
        width: 100%;
        display: flex;
        flex-wrap: wrap;
        padding: 0 10px;
        .lib-item {
          flex: 0 0 50%;
          font-size: 13px;
          background-color: white;
          padding: 5px 10px;
          line-height: 22px;
          cursor: pointer;
          border-radius: 4px;
          margin-bottom: 13px;

          .lib-item-icon {
            display: inline-block;
            color: #999;
            margin-right: 8px;
          }

          &:hover {
            background-color: #f7f8fa;
          }
        }
      }
    }

    .body-content {
      padding: 15px;
      flex: 1;
      height: 100%;

      .body-content-warp {
        max-width: 900px;
        margin: 0 auto;
        height: 100%;
        box-shadow: 0 2px 12px 0 rgba($color: #000, $alpha: 0.1);
        border-radius: $xr-border-radius-base;
        overflow: hidden;
        background-color: white;
        .el-header {
          width: 100%;
          // border-bottom: 1px solid $xr-border-line-color;
          display: flex;
          justify-content: space-between;
          align-items: center;
          .title {
            font-size: 16px;
            font-weight: 600;
            color: #333;
          }
        }
        .body-content-main {
          width: 100%;
          height: 100%;
          overflow-y: auto;
          padding: 10px 16px 30px;
          .el-main {
            .no-list {
              margin: 200px 0;
              color: #ccc;
              @include center;
            }
          }
        }
      }
    }

    .body-right {
      width: 280px;
      max-width: 280px;
      height: 100%;
      overflow-y: auto;
      background-color: white;
    }
  }
}
</style>
