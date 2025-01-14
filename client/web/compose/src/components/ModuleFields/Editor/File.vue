<template>
  <b-form-group
    :label-cols-md="horizontal && '5'"
    :label-cols-xl="horizontal && '4'"
    :content-cols-md="horizontal && '7'"
    :content-cols-xl="horizontal && '8'"
    :class="formGroupStyleClasses"
  >
    <template
      #label
    >
      <div
        v-if="!valueOnly"
        class="d-flex align-items-center text-primary p-0"
      >
        <span
          :title="label"
          class="d-inline-block mw-100"
        >
          {{ label }}
        </span>

        <c-hint :tooltip="hint" />

        <slot name="tools" />
      </div>
      <div
        class="small text-muted"
        :class="{ 'mb-1': description }"
      >
        {{ description }}
      </div>
    </template>

    <div class="d-flex justify-content-between gap-3">
      <uploader
        ref="uploader"
        :endpoint="endpoint"
        :accepted-files="mimetypes"
        :max-filesize="maxSize"
        :form-data="uploaderFormData"
        class="flex-grow-1"
        @uploaded="appendAttachment"
      />

      <b-button
        variant="light"
        class="d-flex align-items-center gap-1"
        @click="openCamera"
      >
        <font-awesome-icon
          class="text-primary"
          :icon="['fas', 'camera']"
        />
      </b-button>
    </div>

    <list-loader
      kind="record"
      :set.sync="set"
      :namespace="namespace"
      :enable-order="field.isMulti"
      enable-delete
      mode="list"
      class="mt-2"
    />

    <b-modal
      ref="modal"
      size="lg"
      modal-class="video-modal-width"
      :title="$t('editor.file.webcam.title')"
      ok-only
      ok-title="Close"
      body-class="d-flex flex-column align-items-center"
      @ok="closeCamera"
      @show="initializeWebcam"
    >
      <video
        ref="video"
        width="640"
        height="480"
        autoplay
        style="display:none;"
      />
      <canvas
        ref="canvas"
        width="640"
        height="480"
        style="display:none;"
      />
      <template #modal-footer>
        <div class="d-flex align-items-center gap-2">
          <b-button
            ref="closeButton"
            variant="light"
            @click="closeCamera"
          >
            {{ $t('editor.file.webcam.buttons.cancel') }}
          </b-button>

          <b-button
            v-if="!hasCapturedImage"
            ref="captureButton"
            style="display:none;"
            variant="primary"
            @click="capturePhoto"
          >
            {{ $t('editor.file.webcam.buttons.capture') }}
          </b-button>
          <b-button
            v-if="hasCapturedImage"
            ref="uploadButton"
            variant="primary"
            style="display:none;"
            @click="uploadCapturedImage"
          >
            {{ $t('editor.file.webcam.buttons.confirm') }}
          </b-button>
        </div>
      </template>
    </b-modal>
    <errors :errors="errors" />
  </b-form-group>
</template>
<script>
import base from './base'
import Uploader from 'corteza-webapp-compose/src/components/Public/Page/Attachment/Uploader'
import ListLoader from 'corteza-webapp-compose/src/components/Public/Page/Attachment/ListLoader'
import { NoID } from '@cortezaproject/corteza-js'

export default {
  i18nOptions: {
    namespaces: 'general',
  },

  components: {
    Uploader,
    ListLoader,
  },

  extends: base,

  data () {
    return {
      video: null,
      canvas: null,
      stream: null,
      capturedImage: null,
      hasCapturedImage: false,
    }
  },

  computed: {
    endpoint () {
      const { moduleID, recordID } = this.record
      const { namespaceID } = this.namespace

      return this.$ComposeAPI.recordUploadEndpoint({
        namespaceID,
        moduleID,
        recordID,
        fieldName: this.field.name,
      })
    },

    uploaderFormData () {
      const fd = {
        fieldName: this.field.name,
      }

      if (this.record && this.record.recordID !== NoID) {
        fd.recordID = this.record.recordID
      }

      return fd
    },

    mimetypes () {
      const a = (this.field.options.mimetypes || '').trim()
      if (!a) {
        return this.$s('compose.Record.Attachments.Mimetypes', ['*/*'])
      }

      return a.split(',').map(p => p.trim())
    },

    maxSize () {
      return this.field.options.maxSize || this.$s('compose.Record.Attachments.MaxSize', 100)
    },

    set: {
      get () {
        return this.field.isMulti ? this.value : [this.value]
      },

      set (v) {
        if (this.field.isMulti) {
          this.value = v
        } else {
          this.value = (Array.isArray(v) && v.length > 0) ? v[0] : undefined
        }
      },
    },
  },

  methods: {
    appendAttachment ({ attachmentID } = {}) {
      if (this.field.isMulti) {
        this.value.push(attachmentID)
      } else {
        this.value = attachmentID
      }
    },
    async initializeWebcam () {
      await this.$nextTick()

      this.video = this.$refs.video
      this.canvas = this.$refs.canvas
      this.captureButton = this.$refs.captureButton
      this.uploadButton = this.$refs.uploadButton
      this.closeButton = this.$refs.closeButton

      this.startWebcam()
    },

    startWebcam () {
      // Get access to the camera
      navigator.mediaDevices.getUserMedia({ video: true })
        .then(stream => {
          this.stream = stream
          this.video.srcObject = stream
          this.video.style.display = 'block'
          this.closeButton.style.display = 'inline-block'

          if (this.hasCapturedImage) {
            this.uploadButton.style.display = 'inline-block'
          } else {
            this.captureButton.style.display = 'inline-block'
          }
        })
        .catch(err => {
          console.error('Error accessing the camera: ' + err)
        })
    },

    openCamera () {
      this.$refs.modal.show()
    },
    capturePhoto () {
      const video = this.$refs.video
      const canvas = document.createElement('canvas')
      canvas.width = video.videoWidth
      canvas.height = video.videoHeight
      canvas.getContext('2d').drawImage(video, 0, 0, canvas.width, canvas.height)

      this.capturedImage = canvas.toDataURL('image/jpeg')
      this.hasCapturedImage = true

      this.stopWebcam()
    },

    uploadCapturedImage () {
      if (this.capturedImage) {
        fetch(this.capturedImage)
          .then(res => res.blob())
          .then(blob => {
            const file = new File([blob], 'webcam-image.jpg', { type: 'image/jpeg' })

            const uploader = this.$refs.uploader
            uploader.$refs.dropzone.addFile(file)
          })

        this.$refs.modal.hide()
      }
    },

    stopWebcam () {
      if (this.stream) {
        this.stream.getTracks().forEach(track => track.stop())
      }
    },
    closeCamera () {
      if (this.hasCapturedImage) {
        this.capturedImage = null
        this.hasCapturedImage = false

        this.startWebcam()
      } else {
        this.stopWebcam()
        this.$refs.modal.hide()
      }
    },
  },
}
</script>

<style>
.video-modal-width .modal-dialog {
  max-width: 700px;
  width: 100%;
}
</style>
