<template>
  <div class="container">
    <h1 class="text-gray-600 text-2xl font-bold">
      ありがとうございました
    </h1>
    <div>
      <p>送信が完了しました</p>
      <span class="inline-block mt-3">
        <nuxt-link class="text-gray-500 font-semibold" to="/">
          撮影画面に戻る
        </nuxt-link>
      </span>
    </div>
  </div>
</template>
<script lang="ts">
export default {
  name: '',
  setup() {
    return {}
  },
}
</script>
<style>
.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
  flex-direction: column;
}
</style>
