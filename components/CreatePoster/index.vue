<template>
  <div v-show="value" class="poster-container">
    <div class="dialog-model" @click="$emit('input', false)"></div>
    <div class="poster-content">
      <x-icon type="icon-close" @click.native="$emit('input', false)"></x-icon>
      <div v-if="JSON.stringify(content) !== '{}'" class="poster-wrap is-border">
        <!-- 最终海报图片 -->
        <img v-if="isCompleted" :src="posterUrl">

        <!-- 用于生成海报 -->
        <div v-else ref="poster" class="create">
          <div class="poster-loading" data-html2canvas-ignore>
            <x-icon type="icon-loading"></x-icon>
            <p>&nbsp;海报生成中...</p>
          </div>
          <div class="content-img-wrap align-center">
            <div style="height: 200px"><img :src="content.imgUrl" height="200"></div>
            <div class="time">
              <p class="day align-center">{{ content.time.replace(/(\d{4})-(\d{1,2})-(\d{1,2})/, '$3') }}</p>
              <p>{{ content.time.replace(/(\d{4})-(\d{1,2})-(\d{1,2})/, '$1/$2') }}</p>
            </div>
          </div>
          <h2 class="title">{{ content.title }}</h2>
          <p class="summary">{{ content.summary }}</p>
          <div class="qrcode">
            <div class="left">
              <p>
                <img :src="content.qrcodeLogo" width="20" height="20" class="vertical-middle">
                <span class="vertical-middle"> {{ content.qrcodeText }}</span>
              </p>
              <p class="tips">长按二维码或扫描二维码查看完整内容</p>
            </div>
            <img :src="QRCodeUrl" width="70" height="70" class="qrcode-img">
          </div>
        </div>
      </div>
      <!-- 右边分享 -->
      <div class="poster-share align-center">
        <h2 class="title">分享本文海报</h2>
        <div class="btn">
          <a :href="`https://sns.qzone.qq.com/cgi-bin/qzshare/cgi_qzshare_onekey?url=${$store.state.info.domain}/details/${$route.params.id}&title=${content.title}&summary=${content.summary}&pics=${poster}`" target="_blank">
            <svg-icon iconName="icon-QQkongjian" class="vertical-middle"></svg-icon>
            <span class="vertical-middle">&nbsp;分享到QQ空间</span>
          </a>
        </div>
        <div class="btn">
          <a :href="`https://service.weibo.com/share/share.php?url=${$store.state.info.domain}/details/${$route.params.id}%230-tsina-1-21107-397232819ff9a47a7b7e80a40613cfe1&title=${content.title}&appkey=1343713053&searchPic=true&pic=${poster}#_loginLayer_1473259217614`" target="_blank">
            <svg-icon iconName="icon-xinlang1" class="vertical-middle"></svg-icon>
            <span class="vertical-middle">&nbsp;分享到新浪</span>
          </a>
        </div>
        <div class="btn">
          <a :href="posterUrl" :download="`${content.title}.png`">
            <svg-icon iconName="icon-xiazai" class="vertical-middle"></svg-icon>
            <span class="vertical-middle">&nbsp;下载海报图</span>
          </a>
        </div>
        <p class="tips">(分享到微信或朋友圈请下载海报图)</p>
      </div>
    </div>
  </div>
</template>
<script>
import { mapActions } from 'vuex'
import QRCode from 'qrcode'
import html2canvas from 'html2canvas'
export default {
  name: 'CreatePoster',
  data () {
    return {
      QRCodeUrl: '',
      posterUrl: '',
      isCompleted: false,
      isFirstCreate: true,
      isShow: true,
      poster: ''
    }
  },
  props: {
    content: {
      type: Object,
      default () {
        return {}
      }
    },
    value: {
      type: Boolean,
      default: false
    }
  },
  watch: {
    value (v) {
      this.isFirstCreate && this._createPoster()
    }
  },
  mounted () {
    // 生成二维码
    const createQR = async text => {
      try {
        this.QRCodeUrl = await QRCode.toDataURL(text)
      } catch (error) {
        console.warn(`[createQRcode]: ${error}`)
        this.$message({
          title: error,
          type: 'error'
        })
      }
    }
    createQR(window.location.href)
  },
  methods: {
    ...mapActions(['uploadImage']),
    async _createPoster () {
      // 生成海报
      const canvas = await html2canvas(this.$refs.poster, {
        useCORS: true,
        logging: false
      })
      const formData = new FormData()
      formData.append('file', canvas.toDataURL('image/png'))
      formData.append('postID', this.content.id)
      formData.append('name', `poster-${this.content.id}`)
      formData.append('url', '/wp-content')
      formData.append('mark', 'upload')
      const data = await this.uploadImage({
        requestData: formData
      })
      this.$emit('on-change', data)
      this.poster = this.$store.state.info.domain + data.path
      this.isFirstCreate = false
      this.isCompleted = true
      this.posterUrl = canvas.toDataURL('image/png')
    }
  }
}
</script>
<style lang="scss" scoped>
$padding: 10px;
.poster-container {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 2000;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;

  .dialog-model {
    @extend %dialog-model;
  }

  .poster-content {
    position: relative;
    display: flex;
    box-sizing: border-box;
    width: 650px;
    height: 500px;
    padding: 20px;
    background: var(--color-sub-background);
    border-radius: $border-radius;
  }

  .icon-close {
    position: absolute;
    right: 10px;
    top: 10px;
    cursor: pointer;
  }

  .poster-wrap {
    position: relative;
    display: flex;
    flex-direction: column;
    width: 60%;

    &.is-border {
      border: 1px solid var(--color-border);
    }

    .create {
      display: flex;
      flex-direction: column;
      height: 100%;
    }

    .poster-loading {
      position: absolute;
      top: 0;
      left: 0;
      z-index: 9;
      display: flex;
      justify-content: center;
      align-items: center;
      width: 100%;
      height: 100%;
      background: var(--color-sub-background);
      font-size: 18px;

      ::v-deep .iconfont {
        font-size: 18px;
      }
    }

    .content-img-wrap {
      position: relative;

      .time {
        position: absolute;
        bottom: 20px;
        right: 20px;
        padding: 0 5px 5px;
        background: rgba(0,0,0,.5);
        font-size: 12px;
        color: #fff;
      }

      .day {
        border-bottom: 1px solid var(--color-border);
        font-size: 30px;
        letter-spacing: 4px;
      }
    }

    .title {
      max-height: 50px;
      padding: 20px $padding 10px;
      overflow: hidden;
      font-size: 18px;
      color: var(--color-primary);
    }

    .summary {
      height: 80px;
      padding: 0 $padding;
      overflow: hidden;
    }
  }

  .qrcode {
    display: flex;
    align-items: center;
    height: 70px;
    margin-top: 10px;
    padding: 10px $padding;
    border-top: 1px dashed var(--color-border);

    .left {
      flex: 1;
    }

    .tips {
      margin-top: 10px;
      font-size: 12px;
    }
  }

  .poster-share {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    flex: 1;
    margin-left: 20px;

    .title {
      margin-top: 50px;
    }

    .btn {
      width: 125px;
      margin-top: 15px;
      padding: 8px 10px;
    }

    .tips {
      margin-top: 10px;
      font-size: 12px;
    }
  }
}
</style>
