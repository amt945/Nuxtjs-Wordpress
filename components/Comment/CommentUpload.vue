<template>
  <div class="upload-img-wrap">
    <div class="sub-upload-wrap align-center">
      <template v-if="!bShowDragWrap">
        <h2 class="title">插入图片</h2>
        <x-icon type="icon-close" @click.native.stop="_hideUpload"></x-icon>
        <div class="progress-wrap">
          <p class="text">上传进度：</p>
          <div class="current-progress">
            <div class="current" :style="`width:${currentProgress}%`"></div>
          </div>
          <div>{{ currentProgress }}%</div>
        </div>
        <div class="select-img">
          <input
            type="file" name="file" value="" ref="inpFile" accept="image/png,image/gif,image/jpeg"
            @change.stop="_preview($event)"
          >
          <p class="mask">
            <span v-if="bFileMark"><x-icon type="icon-upload-img2"></x-icon>点击选择图片或者拖动图片到此窗口内</span>
            <template v-else>
              已选择：<img :src="previewUrl" alt="">
            </template>
          </p>
        </div>
        <div class="btn-wrap">
          <div v-show="!bFileMark" class="btn btn-upload" @click.stop="_uploadImg">上传文件</div>
          <div class="btn btn-insert" @click.stop="_insertImg">插入到文章</div>
        </div>
        <div v-if="resultImgUrl" class="result-img">
          <img :src="resultImgUrl">
        </div>
      </template>
      <div v-else class="drag-wrap">
        <h2 class="title">松开鼠标完成上传</h2>
      </div>
    </div>
    <!-- 遮罩层 -->
    <div class="dialog-model" @click.stop="_hideUpload"></div>
  </div>
</template>
<script>
import { mapState, mapActions } from 'vuex'

export default {
  name: 'CommentUpload',
  data: () => ({
    currentProgress: 0,
    resultImgUrl: '',
    bFileMark: true,
    resFileName: '',
    bShowDragWrap: false,
    previewUrl: ''
  }),
  computed: {
    ...mapState({
      contentUrl: state => state.info.contentUrl,
      templateUrl: state => state.info.templeteUrl
    })
  },
  mounted() {
    // 拖拽上传文件
    const oUploadWrap = document.querySelector('.sub-upload-wrap')
    oUploadWrap.ondragenter = (e) => {
      e.preventDefault()
      this.bShowDragWrap = true
    }
    oUploadWrap.ondragover = (e) => {
      e.preventDefault()
    }
    oUploadWrap.ondrop = (e) => {
      e.preventDefault()
      const oReader = new FileReader()
      const oFile = e.dataTransfer.files[0]
      this.bShowDragWrap = false
      // 判断文件是会否为图片格式
      if (oFile.type.indexOf('image') === -1) {
        this.$message({
          title: '请上传正确的图片格式',
          type: 'warning'
        })
        this.bFileMark = true
      } else {
        oReader.readAsDataURL(oFile)
        oReader.onload = async () => {
          const data = new FormData()
          this.previewUrl = oReader.result
          this.bFileMark = false
          data.append('postID', this.$route.params.id)
          data.append('file', oReader.result)
          data.append('url', this.contentUrl)
          data.append('mark', 'upload')
          this.handleUpload(data)
        }
      }
    }
  },
  methods: {
    ...mapActions(['uploadImage', 'deleteImage']),

    async handleUpload(requestData) {
      try {
        // 上传实时进度
        const config = {
          onUploadProgress: progressEvent => (this.currentProgress = progressEvent.loaded / progressEvent.total * 100)
        }
        const { data } = await this.uploadImage({
          requestData,
          config
        })
        if (!data.code) {
          this.$message({
            title: '上传失败！',
            type: 'error'
          })
          this.currentProgress = 0
        } else {
          this.resultImgUrl = data.path
          this.resFileName = data.name
        }
      } catch (error) {
        if (error === 404) {
          this.$message({
            title: '上传失败(404)！',
            type: 'error'
          })
        }
      }
    },

    _preview(event) {
      const oReader = new FileReader()
      oReader.readAsDataURL(event.target.files[0])
      oReader.onload = () => (this.previewUrl = oReader.result)
      this.bFileMark = false
    },

    // 上传图片
    async _uploadImg() {
      const _file = this.$refs.inpFile
      if (_file.value) {
        const data = new FormData()
        if (_file.files[0].size / 1024 > 2048) {
          this.$message({
            title: '请上传小于2M的图片！',
            type: 'error'
          })
        } else {
          data.append('postID', this.$route.params.id)
          data.append('file', _file.files[0])
          data.append('url', this.contentUrl)
          data.append('mark', 'upload')
          this.handleUpload(data)
        }
      }
    },

    // 隐藏上传控件
    async _hideUpload() {
      if (!this.resFileName) {
        this.$emit('on-close')
        return
      }
      const data = new FormData()
      data.append('mark', 'close')
      data.append('url', this.contentUrl)
      data.append('postID', this.$route.params.id)
      data.append('fileName', this.resFileName)
      // 如果本次上传的图片未发表就从服务器删除此图片
      try {
        await this.deleteImage(data)
      } catch (error) {
        this.$message({
          title: error,
          type: 'error'
        })
      }
      // 关闭控件
      this.$emit('on-close', {
        close: false,
        resFileName: this.resFileName
      })
      this.currentProgress = 0
      this.resultImgUrl = ''
      this.bFileMark = true
    },

    // 插入到文章
    _insertImg() {
      this.$emit('on-confirm', ` [img]${this.resultImgUrl}[/img] `)
      this.$emit('on-close', {
        close: false,
        resFileName: this.resFileName
      })
      this.currentProgress = 0
      this.resultImgUrl = ''
      this.bFileMark = true
    }
  }
}
</script>
<style lang="scss" scoped>
// 评论上传图片
.upload-img-wrap {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 2000;
  width: 100%;
  height: 100%;

  .dialog-model {
    @extend %dialog-model;
  }

  .sub-upload-wrap {
    box-sizing: border-box;
    position: absolute;
    top: 50%;
    left: 50%;
    width: 70%;
    padding: var(--base-gap);
    border-radius: $border-radius;
    background: var(--color-sub-background);
    transform: translate(-50%, -50%);
  }

  // 拖拽上传容器
  .drag-wrap {
    height: 200px;
    border: 2px dashed var(--color-border);
    line-height: 200px;
  }

  .title {
    font-size: 20px;
    font-weight: lighter;
  }

  .icon-close {
    position: absolute;
    top: 10px;
    right: 10px;
    cursor: pointer;
  }

  // 进度条
  .progress-wrap {
    display: flex;
    align-items: center;
  }

  .current-progress {
    flex: 1;
    margin-right: var(--base-gap);
    overflow: hidden;
    background: var(--color-main-background);
    border-radius: $border-radius;

    .current {
      width: 0;
      height: 5px;
      background-color: $color-theme;
      background-image: -webkit-linear-gradient(45deg, rgba(255, 255, 255, .3) 25%, transparent 25%, transparent 50%, rgba(255, 255, 255, .3) 50%, rgba(255, 255, 255, .3) 75%, transparent 75%, transparent);
    }
  }

  // 选择图片
  .select-img {
    position: relative;
    height: 50px;
    margin: var(--base-gap) 0;
    border: 2px dashed var(--color-border);
    line-height: 50px;

    .mask,
    .iconfont {
      font-size: 18px;
    }

    .mask {

      span {
        overflow: hidden;
        display: block;
        text-overflow: ellipsis;
        white-space: nowrap;

        &.inline-block {
          display: inline-block;
          vertical-align: middle;
        }
      }

      img {
        height: 50px;
        vertical-align: top;
      }
    }

    input {
      opacity: 0;
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      cursor: pointer;
    }
  }

  .result-img {
    margin-top: var(--base-gap);
    img {
      max-width: 300px;
      max-height: 350px;
    }
  }

  .btn-wrap {
    display: flex;
    justify-content: center;
    margin-top: var(--base-gap);

    .btn {
      border: 0;
      border-radius: $border-radius;
      background: $color-theme;
      color: $color-white;
      cursor: pointer;

      & + .btn {
        margin-left: var(--base-gap);
      }
    }
  }
}

@media screen and (max-width: 767px) {
  .upload-img-wrap {
    .sub-upload-wrap {
      width: 90%;
    }

    .progress-wrap {
      flex-wrap: wrap;

      .text {
        width: 100%;
        text-align: left;
      }
    }

    .result-img {
      img {
        max-width: 200px;
      }
    }
  }
}

@media screen and (max-height: 500px) {
  .upload-img-wrap {
    .result-img {
      img {
        max-width: 100px;
      }
    }
  }
}
</style>
