<template>
  <b-button
    variant="light"
    class="d-flex align-items-center"
    @click.prevent="$emit('openWebcamModal')"
  >
    <slot name="camera-icon" />
  </b-button>
</template>

<script>
export default {
  name: 'CWebcamButton',
}
</script>
