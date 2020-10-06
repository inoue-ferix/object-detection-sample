<template>
  <div>
    モデル読み込みまで数秒かかります。
    <video ref="video" width="0%" height="240" autoplay></video>
    <canvas ref="canvas"></canvas>
    <div ref="errorMsg"></div>
    検出されたもの : {{ detectedObejct }}
  </div>
</template>
<script lang="ts">
import { ref, onMounted, computed } from '@vue/composition-api'
import axios from 'axios'
import * as tf from '@tensorflow/tfjs'
import * as cocoSsd from '@tensorflow-models/coco-ssd'

export default {
  name: '',
  head() {
    // return {
    //   script: [
    //     {
    //       src: 'https://cdn.jsdelivr.net/npm/@tensorflow/tfjs',
    //     },
    //   ],
    // }
  },
  setup(_props: {}, context: any) {
    const video = ref<HTMLVideoElement>(document.createElement('video'))
    const detectedObejct = ref<string[]>([])
    onMounted(() => {
      // テンソルをキャンバスに描画
      const renderToCanvas = async (ctx: any, a: any) => {
        const [height, width] = a.shape
        const imageData = new ImageData(width, height)
        const data = await a.data()
        for (let i = 0; i < height * width; ++i) {
          const j = i * 4
          const k = i * 3
          imageData.data[j + 0] = data[k + 0]
          imageData.data[j + 1] = data[k + 1]
          imageData.data[j + 2] = data[k + 2]
          imageData.data[j + 3] = 255
        }
        ctx.putImageData(imageData, 0, 0)
      }

      // バウンディングボックスの描画
      const drawBBox = (ctx: any, bbox: any, name: any) => {
        // 枠の描画
        ctx.strokeStyle = 'red'
        ctx.fillStyle = 'red'
        ctx.strokeRect(bbox[0], bbox[1], bbox[2], bbox[3])
        ctx.fillRect(bbox[0], bbox[1] - 20, bbox[2], 20)

        // 名前の描画
        ctx.fillStyle = 'white'
        ctx.font = 'bold 20px sans-serif'
        ctx.textAlign = 'left'
        ctx.textBaseline = 'top'
        ctx.fillText(name, bbox[0] + 8, bbox[1] - 20, bbox[2])
      }

      // 物体検出の開始
      const startDetect = () => {
        cocoSsd.load().then((model) => {
          const webcamElement = context.refs.video
          window.requestAnimationFrame(onFrame.bind(null, model, webcamElement))
        })
      }

      // フレーム毎に呼ばれる
      const onFrame = async (model: any, webcamElement: any) => {
        detectedObejct.value = []
        // 画像分類
        const tensor = tf.browser.fromPixels(webcamElement)
        const predictions = await model.detect(tensor)

        // キャンバスの準備
        const canvas = context.refs.canvas
        const [height, width] = tensor.shape
        canvas.width = width
        canvas.height = height

        // キャンバスの描画
        const ctx = canvas.getContext('2d')
        await renderToCanvas(ctx, tensor)
        for (let i = 0; i < predictions.length; i++) {
          drawBBox(ctx, predictions[i].bbox, predictions[i].class)
          detectedObejct.value.push(predictions[i].class)
        }

        // 次フレーム
        setTimeout(() => {
          window.requestAnimationFrame(onFrame.bind(null, model, webcamElement))
        }, 10000)
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
            startDetect()
          })
          .catch((error) => {
            console.log(error)
          })
      }
    })
    return { detectedObejct }
  },
}
</script>
