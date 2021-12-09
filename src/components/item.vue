<template>
  <div class="item">
    {{ info }}
  </div>
</template>

<script>
export default {
  name: 'Item',
  props: {
    info: [String, Number]
  }
}
</script>

<style lang="scss" scoped>
.item {
  width: 100%;
  height: 100%;
  border: 1px solid #632574;
}
</style>
