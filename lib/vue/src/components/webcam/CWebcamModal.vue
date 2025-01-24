<template>
  <b-modal
    ref="modal"
    size="lg"
    centered
    :title="modalTitle"
    :body-class="processingWebcam ? 'd-flex flex-column align-items-center' : 'p-0'"
    @show="initializeWebcam"
  >
    <b-spinner
      v-show="processingWebcam"
      variant="primary"
    />

    <div
      v-if="showErrorMessage"
      class="bg-danger rounded p-2 m-2 text-white"
    >
      {{ cameraErrorMessage }}
    </div>

    <div
      v-show="!processingWebcam"
      class="embed-responsive embed-responsive-4by3"
    >
      <video
        ref="video"
        autoplay
      />
    </div>
    <template #modal-footer>
      <div class="d-flex align-items-center gap-2">
        <b-button
          ref="closeButton"
          variant="light"
          @click="closeCamera"
        >
          {{ cancelButtonLabel }}
        </b-button>

        <b-button
          :disabled="processingWebcam"
          variant="primary"
          @click="() => (hasCapturedImage ? uploadCapturedImage() : capturePhoto())"
        >
          {{ hasCapturedImage
            ? confirmButtonLabel
            : captureButtonLabel }}
        </b-button>
      </div>
    </template>
  </b-modal>
</template>

<script>
export default {
  name: 'CWebcamModal',

  props: {
    modalTitle: {
      type: String,
      required: true,
    },
    cancelButtonLabel: {
      type: String,
      required: true,
    },
    confirmButtonLabel: {
      type: String,
      required: true,
    },
    captureButtonLabel: {
      type: String,
      required: true,
    },
    cameraErrorMessage: {
      type: String,
      required: true,
    },
  },

  data () {
    return {
      video: null,
      stream: null,
      capturedImage: null,
      hasCapturedImage: false,
      processingWebcam: true,
      showErrorMessage: false,
    }
  },

  methods: {
    async initializeWebcam () {
      await this.$nextTick()

      this.startWebcam()
    },

    startWebcam () {
      // Get access to the camera
      navigator.mediaDevices.getUserMedia({
        video: {
          facingMode: 'user',
        },
      })
        .then(stream => {
          this.showErrorMessage = false

          this.stream = stream
          this.$refs.video.srcObject = stream

          this.processingWebcam = false
        })
        .catch(err => {
          console.error('Error accessing the camera:', err)

          this.showErrorMessage = true
          this.processingWebcam = false
        })
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
            const imageSuffix = new Date().toISOString().replace(/[:.]/g, '-')
            const file = new File([blob], `webcam-image-${imageSuffix}.jpg`, { type: 'image/jpeg' })

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
