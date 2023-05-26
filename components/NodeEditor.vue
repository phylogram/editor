<template>
  <span
    ref="content"
    contenteditable="true"
    @input="updateNode"
  >{{content}}</span>
</template>

<script>
export default {
  name: "NodeEditor",
  props: ['content', 'elementId'],
  methods: {
    updateNode() {
      this.$emit('input', {
        content: this.$refs.content.innerText,
        elementId: this.elementId})
      ;
    }
  }
}
</script>

<style scoped>

</style>
