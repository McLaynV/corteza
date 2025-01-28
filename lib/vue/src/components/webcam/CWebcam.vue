<template>
  <div>
    <b-button
      v-b-tooltip.noninteractive.hover="{ title: labels.tooltip, container: '#body' }"
      variant="light"
      :class="buttonClass"
      @click.prevent="$emit('openWebcamModal')"
    >
      <slot />
    </b-button>
    <b-modal
      ref="modal"
      size="lg"
      centered
      :title="labels.modalTitle"
      body-class="p-0"
      @show="initializeWebcam"
    >
      <div
        v-if="showErrorMessage"
        class="rounded p-3 m-2 text-danger"
      >
        {{ labels.cameraErrorMessage }}
      </div>

      <div
        v-else
        class="embed-responsive embed-responsive-4by3 d-flex justify-content-center align-items-center"
      >
        <b-spinner
          v-if="processingWebcam"
          variant="primary"
        />

        <video
          v-show="!processingWebcam"
          ref="video"
          autoplay
        />
      </div>

      <template #modal-footer>
        <div class="d-flex align-items-center gap-2">
          <b-button
            ref="closeButton"
            variant="light"
            @click="handleCloseClick"
          >
            {{ labels.cancelButtonLabel }}
          </b-button>

          <b-button
            :disabled="processingWebcam"
            variant="primary"
            @click="handleCaptureClick"
          >
            {{ hasCapturedImage
              ? labels.confirmButtonLabel
              : labels.captureButtonLabel }}
          </b-button>
        </div>
      </template>
    </b-modal>
  </div>
</template>

<script>
export default {
  name: 'CWebcamModal',

  props: {
    buttonClass: {
      type: String,
      default: '',
    },
    labels: {
      type: Object,
      default: () => ({}),
    },
  },

  data () {
    return {
      video: null,
      stream: null,
      capturedImage: null,
      processingWebcam: true,
      showErrorMessage: false,
    }
  },

  computed: {
    handleCaptureClick () {
      return () => this.hasCapturedImage ? this.uploadCapturedImage() : this.capturePhoto()
    },

    handleCloseClick () {
      return () => this.hasCapturedImage ? this.discardCapturedImage() : this.closeCamera()
    },

    hasCapturedImage () {
      return !!this.capturedImage
    },
  },

  methods: {
    async initializeWebcam () {
      await this.$nextTick()

      this.startWebcam()
    },

    startWebcam () {
      this.showErrorMessage = false

      // Get access to the camera
      navigator.mediaDevices.getUserMedia({
        video: {
          facingMode: 'user',
        },
      })
        .then(stream => {
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

      this.stopWebcam()
    },

    uploadCapturedImage () {
      if (this.capturedImage) {
        fetch(this.capturedImage)
          .then(res => res.blob())
          .then(blob => {
            const imageSuffix = new Date().toISOString().replace(/[:.]/g, '-')
            const file = new File([blob], `webcam-image-${imageSuffix}.jpg`, { type: 'image/jpeg' })

            this.$emit('uploaded', file)
          })

        this.closeCamera()
      }
    },

    stopWebcam () {
      if (this.stream) {
        this.stream.getTracks().forEach(track => track.stop())
      }
    },

    closeCamera () {
      this.stopWebcam()
      this.$refs.modal.hide()
      this.capturedImage = null
    },

    discardCapturedImage () {
      this.capturedImage = null
      this.startWebcam()
    },
  },
}
</script>
