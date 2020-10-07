<template>
  <div>
    <video ref="video" width="500" autoplay></video>
    <canvas ref="canvas"></canvas>
    <div class="text-lg">
      1秒ごとに差分をチェックしています。<br />
      差の割合: {{ diffNum }} %
    </div>
  </div>
</template>
<script lang="ts">
import { ref, onMounted, computed } from '@vue/composition-api'
import axios from 'axios'

export default {
  name: '',
  setup(_props: {}, context: any) {
    const video = ref<HTMLVideoElement>(document.createElement('video'))
    const ctx = ref<CanvasRenderingContext2D | null>(null)
    const canvas = ref<HTMLCanvasElement>(document.createElement('canvas'))
    const prevArr = ref<any>([])
    const currentArr = ref<any>([])
    const diffNum = ref(0)
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
    onMounted(() => {
      const diff = (data1: number[], data2: number[]) => {
        let allPixelNum = 0
        let diffPixelNum = 0
        for (let i = 0; i <= data1.length; i++) {
          allPixelNum++
          const diffs = Math.abs(data1[i] - data2[i])
          if (diffs > 10) {
            diffPixelNum++
          }
        }
        diffNum.value = Number(((diffPixelNum / allPixelNum) * 100).toFixed(2))
      }
      const startDetect = () => {
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
        currentArr.value = ctx.value.getImageData(
          0,
          0,
          video.value.videoWidth,
          video.value.videoHeight
        ).data
        if (prevArr.value.length === 0) {
          console.log('ffff')
          prevArr.value = currentArr.value
          return
        }
        diff(prevArr.value, currentArr.value)
        prevArr.value = currentArr.value
      }
      // Webカメラの開始
      const constraints = {
        audio: false,
        video: { facingMode: 'environment' }, // アウトカメラを優先的に使う
      }
      video.value = context.refs.video

      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        navigator.mediaDevices
          .getUserMedia(constraints)
          .then((stream: any) => {
            video.value.srcObject = stream
            video.value.play()
            // 次フレーム
            console.log('start')
            setInterval(() => {
              startDetect()
            }, 1000)
          })
          .catch((error) => {
            console.log(error)
          })
      }
    })
    return { diffNum }
  },
}
</script>
