<template>
  <div class="mainpage-container">
    <button @click="handleBtnClick">
      {{ btnText }}
    </button>
    <div style="position: relative">
      <video
        ref="video"
        autoplay
        playsinline
        @loadeddata="predictWebcam"
      ></video>
      <canvas
        ref="canvasElement"
        style="position: absolute; left: 0px; top: 0px; z-index: 1"
      ></canvas>
    </div>
    <div ref="resultsContainerElement" class="resultsContainerElement">
      <!-- <canvas ref="resultCanvasELement" :width="videoWidth" :height="videoHeight"></canvas> -->
    </div>
    <div class="heartRate">
      <p class="title">心率</p>
      <p v-for="(item, i) in heartRateDisplay" :key="i">{{ item }}</p>
    </div>
    <div class="spo2">
      <p class="title">血氧浓度</p>
      <p v-for="(item, i) in spO2Display" :key="i">{{ item }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import {
  FilesetResolver,
  HandLandmarker,
  DrawingUtils,
  type HandLandmarkerResult,
} from "@mediapipe/tasks-vision";
import FFT from "fft.js";

let handLandmarker: HandLandmarker;
const runningMode = "VIDEO";
let webcamRunning: Boolean = false; // 控制是否持续监测
const btnText = ref("打开摄像头");
const videoHeight = 360;
const videoWidth = 480;
const resultHeight = 80;
const resultWidth = 80;
const video = ref<HTMLVideoElement | null>(null);
const canvasElement = ref<HTMLCanvasElement | null>(null);
const resultsContainerElement = ref<HTMLCanvasElement | null>(null);
let canvasCtx: CanvasRenderingContext2D;
const fingerArr = [4, 8, 12, 16, 20]; // 指尖的位置
const resultsCanvasCtx: CanvasRenderingContext2D[] = [];

// Before we can use HandLandmarker class we must wait for it to finish
// loading. Machine Learning models can be large and take a moment to
// get everything needed to run.
const createHandLandmarker = async () => {
  const vision = await FilesetResolver.forVisionTasks(
    "node_modules/@mediapipe/tasks-vision/wasm"
  );
  handLandmarker = await HandLandmarker.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath: "app/shared/models/hand_landmarker.task",
      delegate: "GPU",
    },
    runningMode: runningMode,
    numHands: 1,
  });
};
createHandLandmarker();

// Enable the live webcam view and start detection.
const enableCam = () => {
  if (!handLandmarker) {
    alert("Please wait for handLandmarker to load");
    return;
  }

  if (webcamRunning === true) {
    webcamRunning = false;
    btnText.value = "开启监测";
  } else {
    webcamRunning = true;
    btnText.value = "暂停监测";
  }

  // getUsermedia parameters.
  const constraints = {
    video: true,
  };

  // Activate the webcam stream.
  navigator.mediaDevices.getUserMedia(constraints).then((stream) => {
    if (video.value) video.value.srcObject = stream; // 将画面输出video中
  });
};

// Check if webcam access is supported.
const hasGetUserMedia = computed(() => !!navigator.mediaDevices?.getUserMedia);

const handleBtnClick = () => {
  if (hasGetUserMedia.value) {
    enableCam(); // 打开设想头
  } else {
    console.warn("getUserMedia() is not supported by your browser");
  }
};

// 浏览器页面加载之前会执行这个函数
onMounted(() => {
  // 获取到canvas的上下文
  canvasCtx = canvasElement.value?.getContext("2d") as CanvasRenderingContext2D;
  // resultCanvasCtx = resultCanvasELement.value?.getContext('2d') as CanvasRenderingContext2D;

  // 设置video元素的宽高和canvas一致
  if (canvasElement.value && video.value) {
    canvasElement.value.height = videoHeight;
    video.value.height = videoHeight;
    canvasElement.value.width = videoWidth;
    video.value.width = videoWidth;
  }

  // 创建存放指尖图像结果的canvas
  for (let i = 0; i < 5; i++) {
    // 创建canvas
    const resultCanvasELement = document.createElement("canvas");
    resultCanvasELement.height = resultHeight;
    resultCanvasELement.width = resultWidth;
    if (resultsContainerElement.value) {
      resultsContainerElement.value.appendChild(resultCanvasELement);
    }
    const ctx = resultCanvasELement.getContext(
      "2d"
    ) as CanvasRenderingContext2D;

    // 加入数组
    resultsCanvasCtx.push(ctx);
  }
});

let lastVideoTime = -1;
let results: HandLandmarkerResult;
const predictWebcam = async () => {
  // 当前时间和最后一次时间不一致时，重新检测。实现连续的检测，每次检测更新结果
  let nowInMs = Date.now();
  if (video.value && video.value.currentTime !== lastVideoTime) {
    lastVideoTime = video.value.currentTime;
    results = handLandmarker.detectForVideo(video.value, nowInMs);
  }

  canvasCtx.save();
  canvasCtx.clearRect(0, 0, videoWidth, videoHeight);
  const drawingUtils = new DrawingUtils(canvasCtx);

  if (results.landmarks) {
    for (let landmarks of results.landmarks) {
      landmarks = landmarks.filter((item, i) => fingerArr.includes(i));
      drawingUtils.drawLandmarks(landmarks, {
        // 画红点在五个手指上
        color: "#FF0000",
        lineWidth: 2,
      });

      landmarks.forEach((item, i) => {
        if (item.x <= 1 && item.y <= 1) {
          resultsCanvasCtx[i].save();
          // 这里位置有一定的偏差，暂未解决，暂时加上30px
          resultsCanvasCtx[i].drawImage(
            video.value as CanvasImageSource,
            item.x * videoWidth + 30,
            item.y * videoHeight + 30,
            resultWidth,
            resultHeight,
            0,
            0,
            resultWidth,
            resultWidth
          );

          // 提取当前帧图像的RGB值
          const frame = resultsCanvasCtx[i].getImageData(
            0,
            0,
            resultWidth,
            resultHeight
          );
          extractColorSignals(frame, i); // 加入RGB通道并计算心率和血氧

          resultsCanvasCtx[i].restore();
        } else {
          resultsCanvasCtx[i].clearRect(0, 0, resultWidth, resultHeight);
        }
      });
    }
  } else {
    resultsCanvasCtx.forEach((item) => {
      item.clearRect(0, 0, resultWidth, resultHeight);
    });
  }

  canvasCtx.restore();

  // 持续监测
  if (webcamRunning === true) {
    window.requestAnimationFrame(predictWebcam);
  }
};

// RBG信号
const redSignal: Array<Array<number>> = new Array(5);
const greenSignal: Array<Array<number>> = new Array(5);
const blueSignal: Array<Array<number>> = new Array(5);
for (let i = 0; i < 5; i++) {
  redSignal[i] = new Array();
  greenSignal[i] = new Array();
  blueSignal[i] = new Array();
}
const fps = 30; // 帧率
const heartRateDisplay = ref<number[]>(new Array(5));
const spO2Display = ref<number[]>(new Array(5));

/**
 * 提取RGB数据, 并计算心率和血氧浓度
 * @param frame 当前帧的数据
 * @param index 第几个指尖数据
 */
function extractColorSignals(frame: ImageData, index: number) {
  const length = frame.data.length / 4;
  let red = 0,
    green = 0,
    blue = 0;

  for (let i = 0; i < length; i++) {
    red += frame.data[i * 4];
    green += frame.data[i * 4 + 1];
    blue += frame.data[i * 4 + 2];
  }

  redSignal[index].push(red / length);
  greenSignal[index].push(green / length);
  blueSignal[index].push(blue / length);

  if (redSignal[index].length >= 256) {
    // 为fft保留2的幂次方的length
    // 计算心率
    // 对绿色分量进行 FFT 变换
    const greenSpectrum = computeFFT(greenSignal[index]);
    // 得出心率
    const heartRate = calculateHeartRate(greenSpectrum);
    heartRateDisplay.value[index] = heartRate;

    // 根据红色和蓝色计算血氧浓度
    const spO2 = calculateSpO2(redSignal[index], blueSignal[index]);
    spO2Display.value[index] = spO2;

    redSignal[index].shift();
    greenSignal[index].shift();
    blueSignal[index].shift();
  }
}

// 计算FFT并返回频谱
function computeFFT(data: number[]) {
  const fft = new FFT(data.length);
  const out = fft.createComplexArray();
  fft.realTransform(out, data);
  fft.completeSpectrum(out);

  const spectrum = [];
  for (let i = 0; i < out.length / 2; i += 2) {
    const magnitude = Math.sqrt(out[i] * out[i] + out[i + 1] * out[i + 1]);
    spectrum.push(magnitude);
  }

  return spectrum;
}

// 计算心跳频率
function calculateHeartRate(spectrum: number[]): number {
  // 心率范围的频率（每秒）
  const minBpm = 60; // 最小心率
  const maxBpm = 120; // 最大心率
  const minFreq = minBpm / 60;
  const maxFreq = maxBpm / 60;

  // 计算频率分辨率
  const freqResolution = fps / spectrum.length;

  // 带通滤波器：只保留心率范围内的频率分量
  const filteredSpectrum = spectrum.map((value, index) => {
    const frequency = index * freqResolution;
    if (frequency >= minFreq && frequency <= maxFreq) {
      return value;
    } else {
      return 0;
    }
  });

  // 找到带通滤波后的最大值对应的频率
  const peakIndex = filteredSpectrum.reduce((maxIndex, value, index, array) => {
    return value > array[maxIndex] ? index : maxIndex;
  }, 0);

  // 计算心率
  const frequency = peakIndex * freqResolution;
  const heartRate = frequency * 60; // 转换为每分钟心跳数

  return heartRate;
}

// 计算血氧浓度
function calculateSpO2(redChannelData: number[], blueChannelData: number[]) {
  // 计算平均值
  const redMean =
    redChannelData.reduce((sum, value) => sum + value, 0) /
    redChannelData.length;
  const blueMean =
    blueChannelData.reduce((sum, value) => sum + value, 0) /
    blueChannelData.length;

  // 计算标准偏差
  const redStdDev = Math.sqrt(
    redChannelData.reduce(
      (sum, value) => sum + Math.pow(value - redMean, 2),
      0
    ) / redChannelData.length
  );
  const blueStdDev = Math.sqrt(
    blueChannelData.reduce(
      (sum, value) => sum + Math.pow(value - blueMean, 2),
      0
    ) / blueChannelData.length
  );

  // 计算比值 R
  const R = redStdDev / redMean / (blueStdDev / blueMean);

  // 计算 SpO2
  const SpO2 = 110 - 25 * R;

  return SpO2;
}
</script>

<style scoped>
.resultsContainerElement {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
</style>
