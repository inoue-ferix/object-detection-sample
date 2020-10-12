<template>
  <div class="container">
    <div>
      <h1 class="font-semibold text-3xl m-3 block">カメラテスト</h1>
      <span class="cursor-pointer font-semibold text-gray-600">
        {{ count }} 回撮影済み
      </span>

      <div v-show="isCameraMode" class="mt-4 flex flex-col">
        <video
          id="video"
          ref="video"
          width="100%"
          height="500"
          autoplay
        ></video>
      </div>

      <span class="inline-block">
        {{ message }}
      </span>
      <canvas id="canvas" ref="canvas"></canvas>
      <div v-if="captures.length > 0">
        <label
          class="block text-gray-500 font-bold text-left mb-1 md:mb-0 pr-4"
          for="upload"
        >
          撮影された画像
        </label>
        <img id="upload" :src="captures[0]" />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { ref, onMounted, computed } from '@vue/composition-api'
import axios from 'axios'
export default {
  name: 'Index',
  setup(_props: {}, context: any) {
    const video = ref<HTMLVideoElement>(document.createElement('video'))
    const canvas = ref<HTMLCanvasElement>(document.createElement('canvas'))
    const captures = ref<Array<string>>([])
    const isLoading = ref(false)
    const ctx = ref<CanvasRenderingContext2D | null>(null)
    const message = ref('')
    const isUploaded = ref(false)
    const param = ref('')
    const count = ref(0)
    const isCameraMode = ref(true)
    const toggleMode = () => {
      isCameraMode.value = !isCameraMode.value
    }
    const constraints = {
      audio: false,
      video: { facingMode: 'environment' }, // アウトカメラを優先的に使う
    }

    const snapShotMessage = computed(() => {
      if (captures.value.length > 0) return 'もう一度撮影する'
      return '撮影'
    })
    const send = async () => {
      if (captures.value.length === 0) return

      const data = { data: { image: captures.value[0] } }
      try {
        await axios.post(
          'https://getaijawdl.execute-api.ap-northeast-1.amazonaws.com/dev/image',
          data
        )
        context.root.$router.push('/thankyou')
      } catch (e) {
        alert(`${e.message}`)
      }
    }
    onMounted(() => {
      video.value = context.refs.video

      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        navigator.mediaDevices.getUserMedia(constraints).then((stream: any) => {
          video.value.srcObject = stream
          video.value.play()
          setInterval(() => {
            capture()
            count.value++
          }, 10000)
        })
      }
    })

    const onImagePushed = (imageData: string) => {
      captures.value.unshift(imageData)
      isUploaded.value = true
    }

    const getCanvas = (
      canvasId: string,
      width: number,
      hegiht: number
    ): HTMLCanvasElement => {
      const _canvas = document.createElement(canvasId) as HTMLCanvasElement
      _canvas.height = hegiht
      _canvas.width = width
      return _canvas
    }

    const capture = () => {
      isLoading.value = true
      if (!ctx.value) {
        canvas.value = getCanvas(
          'canvas',
          video.value.videoWidth,
          video.value.videoHeight
        )
        ctx.value = canvas.value.getContext('2d') as CanvasRenderingContext2D
      }
      if (!canvas.value) return
      ctx.value.drawImage(
        video.value,
        0,
        0,
        video.value.videoWidth,
        video.value.videoHeight
      )
      captures.value.unshift(canvas.value.toDataURL('image/jpeg'))
      isLoading.value = false
      message.value = '撮影しました'
      setTimeout(() => {
        message.value = ''
      }, 4000)
    }
    return {
      capture,
      captures,
      onImagePushed,
      isLoading,
      message,
      snapShotMessage,
      send,
      param,
      toggleMode,
      isCameraMode,
      isUploaded,
      count,
    }
  },
}
</script>

<style>
#canvas {
  display: none;
}
#video {
  background-color: #000;
}
html,
body {
  font-family: 'Noto Sans JP', sans-serif;
  font-weight: 400;
}
h1 {
  font-weight: 800;
}

.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
  flex-direction: column;
}

.title {
  font-family: Nato san JP, sans-serif;
  display: block;
  font-weight: 300;
  font-size: 100px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 42px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}
</style>
