<template>
  <div>
    <h1>Dashboard</h1>
    Ini Dashboard Pengurus
  </div>
</template>

<script>
export default {
  data: () => ({
    pengurus: {}
  }),
  mounted() {
    this.pengurus = this.$store.state.pengurus
  }
}
</script>

<style>

</style>