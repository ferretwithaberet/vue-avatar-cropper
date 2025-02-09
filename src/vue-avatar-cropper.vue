<template>
  <div class="avatar-cropper">
    <div
      class="avatar-cropper-overlay"
      :class="{'avatar-cropper-overlay-inline': inline}"
      v-if="dataUrl"
    >
      <div class="avatar-cropper-mark" v-if="!inline">
        <a
          @click="cancel"
          class="avatar-cropper-close"
          :title="labels.cancel"
          href="javascript:;"
        >&times;</a>
      </div>

      <div class="avatar-cropper-container">
        <div class="avatar-cropper-image-container">
          <img
            ref="img"
            :src="dataUrl"
            alt
            @load.stop="createCropper"
            @error="onImgElementError"
          />
        </div>

        <div class="avatar-cropper-footer">
          <button
            @click.stop.prevent="cancel"
            class="avatar-cropper-btn"
          >
            {{ labels.cancel }}
          </button>

          <button
            @click.stop.prevent="submit"
            class="avatar-cropper-btn"
          >
            {{ labels.submit }}
          </button>
        </div>
      </div>
    </div>

    <input
      v-if="!file"
      :accept="cleanedMimes"
      :capture="capture"
      class="avatar-cropper-img-input"
      ref="input"
      type="file"
      @change="onFileInputChange"
    />
  </div>
</template>

<script>
import 'cropperjs/dist/cropper.css'
import Cropper from 'cropperjs'

export default {
  name: 'AvatarCropper',

  model: {
    prop: 'trigger',
    event: 'triggered',
  },

  props: {
    trigger: {
      type: Boolean,
      default: false,
    },

    file: {
      type: File,
    },

    uploadHandler: {
      type: Function,
    },

    uploadUrl: {
      type: String,
    },

    requestMethod: {
      type: String,
      default: 'POST',
    },

    uploadHeaders: {
      type: Object,
    },

    uploadFormName: {
      type: String,
      default: 'file',
    },

    uploadFormData: {
      type: Object,
      default() {
        return {}
      },
    },

    cropperOptions: {
      type: Object,
      default() {
        return {
          aspectRatio: 1,
          autoCropArea: 1,
          viewMode: 1,
          movable: false,
          zoomable: false,
        }
      },
    },

    outputOptions: {
      type: Object,
    },

    outputMime: {
      type: String,
      default: null,
    },

    outputQuality: {
      type: Number,
      default: 0.9,
    },

    mimes: {
      type: String,
      default: 'image/png, image/gif, image/jpeg, image/bmp, image/x-icon',
    },

    capture: {
      type: String,
    },

    labels: {
      type: Object,
      default() {
        return {
          submit: '提交',
          cancel: '取消',
        }
      },
    },

    withCredentials: {
      type: Boolean,
      default: false,
    },

    inline: {
      type: Boolean,
      default: false,
    },
  },

  data() {
    return {
      cropper: undefined,
      dataUrl: undefined,
      filename: undefined,
    }
  },

  computed: {
    cleanedMimes() {
      if (!this.mimes) throw new Error('vue-avatar-cropper: mimes prop cannot be empty')

      return this.mimes.trim().toLowerCase()
    },
  },

  watch: {
    trigger(value) {
      if (!value) return

      if (this.file) {
        this.onFileChange(this.file)
      } else {
        this.pickImage()
      }

      this.$emit('triggered', false)
    },
  },

  mounted() {
    this.$emit('triggered', false)
  },

  methods: {
    destroy() {
      if (this.cropper) this.cropper.destroy()

      if (this.$refs.input) this.$refs.input.value = ''

      this.dataUrl = undefined
    },

    submit() {
      this.$emit('submit')

      if (this.uploadUrl) {
        this.uploadImage()
      } else if (this.uploadHandler) {
        this.uploadHandler(this.cropper)
      } else {
        this.$emit('error', 'No upload handler found.', 'user')
      }

      this.destroy()
    },

    cancel(){
        this.$emit('cancel')
        this.destroy()
    },

    onImgElementError() {
      this.$emit('error', 'File loading failed.', 'load')
      this.destroy()
    },

    pickImage() {
      if (this.$refs.input) this.$refs.input.click()
    },

    onFileChange(file) {
      if (this.cleanedMimes === 'image/*') {
        if (file.type.split('/')[0] !== 'image') {
          this.$emit('error', 'File type not correct.', 'user')
          return
        }
      } else if (this.cleanedMimes) {
        const correctType = this.cleanedMimes.split(', ').find((mime) => mime === file.type)

        if (!correctType) {
          this.$emit('error', 'File type not correct.', 'user')
          return
        }
      }

      const reader = new FileReader()
      reader.onload = (e) => {
        this.dataUrl = e.target.result
      }

      reader.readAsDataURL(file)

      this.filename = file.name || 'unknown'
      this.mimeType = this.mimeType || file.type
      this.$emit('changed', file, reader)
    },

    onFileInputChange(e) {
      if (!e.target.files || !e.target.files[0]) return

      this.onFileChange(e.target.files[0])
    },

    createCropper() {
      this.cropper = new Cropper(this.$refs.img, this.cropperOptions)
    },

    uploadImage() {
      this.cropper.getCroppedCanvas(this.outputOptions).toBlob(
        (blob) => {
          const form = new FormData()
          const xhr = new XMLHttpRequest()
          const data = Object.assign({}, this.uploadFormData)

          xhr.withCredentials = this.withCredentials

          for (const key in data) {
            form.append(key, data[key])
          }

          form.append(this.uploadFormName, blob, this.filename)

          this.$emit('uploading', form, xhr)

          xhr.open(this.requestMethod, this.uploadUrl, true)

          for (const header in this.uploadHeaders) {
            xhr.setRequestHeader(header, this.uploadHeaders[header])
          }

          xhr.onreadystatechange = () => {
            if (xhr.readyState === 4) {
              let response = ''

              try {
                response = JSON.parse(xhr.responseText)
              } catch (err) {
                response = xhr.responseText
              }

              this.$emit('completed', response, form, xhr)

              if ([200, 201, 204].indexOf(xhr.status) > -1) {
                this.$emit('uploaded', response, form, xhr)
              } else {
                this.$emit('error', 'Image upload fail.', 'upload', xhr)
              }
            }
          }

          xhr.send(form)
        },
        this.outputMime,
        this.outputQuality,
      )
    },
  },
}
</script>

<style lang="scss">
.avatar-cropper {
  .avatar-cropper-overlay {
    text-align: center;
    display: flex;
    align-items: center;
    justify-content: center;
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 99999;
  }

  .avatar-cropper-overlay-inline{
    position: initial;
  }

  .avatar-cropper-img-input {
    display: none;
  }

  .avatar-cropper-close {
    float: right;
    padding: 20px;
    font-size: 3rem;
    color: #fff;
    font-weight: 100;
    text-shadow: 0px 1px rgba(40, 40, 40, 0.3);
  }

  .avatar-cropper-mark {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.1);
  }

  .avatar-cropper-container {
    background: #fff;
    z-index: 999;
    box-shadow: 1px 1px 5px rgba(100, 100, 100, 0.14);

    .avatar-cropper-image-container {
      position: relative;
      max-width: 400px;
      height: 300px;
    }

    img {
      max-width: 100%;
      height: 100%;
    }

    .avatar-cropper-footer {
      display: flex;
      align-items: stretch;
      align-content: stretch;
      justify-content: space-between;

      .avatar-cropper-btn {
        width: 50%;
        padding: 15px 0;
        cursor: pointer;
        border: none;
        background: transparent;
        outline: none;

        &:hover {
          background-color: #2aabd2;
          color: #fff;
        }
      }
    }
  }
}
</style>
