<template>
  <div data-class="swipe-item">
    <slot></slot>
  </div>
</template>
<style scoped>
  /*防止对自定义设置class="{{clazz}}"时被覆盖，所以这里不能使用class而是使用自定义属性data-class*/
  [data-class] {
    height: 100%;
    width: 33.33%;
    position: absolute;
    display: none;
  }
</style>
<script>
  export default{
    name: 'mt-swipe-item',

    ready() {
      this.$dispatch('swipeItemCreated', this);
    },

    detached() {
      this.$dispatch('swipeItemDestroyed', this);
    },

    destroyed() {
      this.$dispatch('swipeItemDestroyed', this);
    }
  }
</script>
