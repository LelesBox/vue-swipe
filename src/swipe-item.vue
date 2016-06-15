<template>
  <div class="swipe-item">
    <slot></slot>
  </div>
</template>
<style scoped>
  .swipe-item {
    height: 100%;
    width: 33.33%;
    position: absolute;
    display: none;
  }
  .swipe-item.active {
    display: block;
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
