<template>
  <div>
    <TheLoading v-if="isLoading" />
    <input
      type="file"
      accept="image/*"
      class="border-gray-300 p-3"
      @change="onFileChange($event)"
    />
  </div>
</template>
<script lang="ts">
import { ref, defineComponent, SetupContext, watch } from '@vue/composition-api'
export default defineComponent({
  name: '',
  setup(_props: {}, { emit }: SetupContext) {
    const imageData = ref('')
    const isLoading = ref(false)
    const onFileChange = (e: any) => {
      isLoading.value = true
      const files = e.target.files

      if (files.length > 0) {
        const file = files[0]
        const reader = new FileReader()
        reader.onload = (e: any) => {
          imageData.value = e.target.result
          isLoading.value = false
        }
        reader.readAsDataURL(file)
      }
    }
    watch(
      () => imageData.value,
      () => {
        emit('onImagePushed', imageData.value)
      }
    )

    return {
      onFileChange,
      imageData,
      isLoading,
    }
  },
})
</script>
