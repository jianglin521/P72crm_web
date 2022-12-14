<template>
  <div ref="wkSignaturePad" class="wk-signature-pad">
    <wk-signature-image
      v-if="disabled"
      :src="value.url"
      :height="height"
    />
    <template v-else>
      <vue-signature-pad
        ref="signaturePad"
        :key="height"
        :options="options"
        :height="height"
        width="100%"
      />
      <div class="wk-signature-pad__handle">
        <el-button type="text" icon="wk wk-icon-reply" @click="handleClick('undo')">撤回</el-button>
        <el-button type="text" icon="wk wk-icon-bin" @click="handleClick('clear')">清空</el-button>
      </div>
    </template>
  </div>
</template>

<script>

import { crmFileSingleSaveAPI } from '@/api/common'

import VueSignaturePad from './VueSignaturePad'
import WkSignatureImage from './Image'

import { valueEquals } from 'element-ui/lib/utils/util'
import { getImageData } from '@/utils'


export default {
  // 签名
  name: 'WkSignaturePad',

  components: {
    VueSignaturePad,
    WkSignatureImage
  },

  props: {
    value: [Object, String], // batchId 交互
    data: String, // 同步数据源
    disabled: Boolean
  },

  data() {
    return {
      options: {
        backgroundColor: '#fff',
        onEnd: this.onEnd,
        minWidth: 1,
        maxWidth: 3
      },
      height: '150px'
    }
  },

  computed: {
  },

  watch: {
    data(newVal, oldVal) {
      if (!valueEquals(newVal, oldVal)) {
        this.$refs.signaturePad.fromDataURL(newVal)
      }
    }
  },

  created() {},

  mounted() {
    if (this.value) {
      this.getData()
    } else {
      this.$emit('input', {})
    }

    this.height = `${parseInt(this.$refs.wkSignaturePad.clientWidth / 2.6)}px`
  },

  beforeDestroy() {},

  methods: {
    getData() {
      getImageData(this.value.url)
        .then(data => {
          var img = new Image()
          img.onload = () => {
            const canvasWidth = this.$refs.signaturePad.canvas.width
            const width = img.width
            const ratio = canvasWidth / width
            console.log(width, canvasWidth, ratio)
            this.options.minWidth = this.options.minWidth * ratio
            this.options.maxWidth = this.options.maxWidth * ratio

            this.$nextTick(() => {
              this.$refs.signaturePad.fromDataURL(data.src)
            })
          }
          img.src = data.src
        })
        .catch(() => {
        })
    },

    onEnd() {
      const { isEmpty, data } = this.$refs.signaturePad.saveSignature()
      this.$emit('update:data', data)
      if (!isEmpty) {
        this.$refs.signaturePad.toBlob((blob) => {
          this.uploadFile(blob)
        })
      }
    },

    uploadFile(blob) {
      crmFileSingleSaveAPI({
        file: blob,
        batchId: this.value
      }).then((res) => {
        this.$emit('input', res.data)
      }).catch(() => {})
    },

    handleClick(type) {
      if (type === 'clear') {
        this.$refs.signaturePad.clearSignature()
      } else if (type === 'undo') {
        this.$refs.signaturePad.undoSignature()
      }
    }
  }
}
</script>
<style lang="scss">
.el-form-item.is-error {
  .wk-signature-pad {
    border-color: #f56c6c;
  }
}
</style>

<style lang="scss" scoped>
.wk-signature-pad {
  position: relative;
  border-radius: 4px;
  border: 1px solid #dcdfe6;
  overflow: hidden;

  &__handle {
    position: absolute;
    right: 15px;
    bottom: 0;
    .el-button--text {
      color: #999;
      &:hover {
        color: $xr-color-primary;
      }
    }
  }
}
</style>

