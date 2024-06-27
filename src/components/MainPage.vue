<template>
  <div class="greetings" id="demos">
    <h2><br>Demo: Webcam continuous hand gesture detection</h2>
    <p>Use your hand to make gestures in front of the camera to get gesture classification. </br>Click <b>enable webcam</b> below and grant access to the webcam if prompted.</p>

    <div id="liveView" class="videoView">
      <button class="mdc-button mdc-button--raised" @click="handleEnabelWebcamButtonClick">
        {{ btnText }}
      </button>
      <!-- <button @click="stopWebcam">关闭摄像头</button> -->
      <div style="position: relative;">
        <video ref="video" autoplay playsinline @loadeddata="predictWebcam"></video>
        <canvas class="output_canvas" ref="canvasElement" width="1280" height="720" style="position: absolute; left: 0px; top: 0px;"></canvas>
        <p ref='gestureOutput' class="output"></p>
      </div>
    </div>

    <div class="fingers" style="margin-top: 500;">
      <!-- <canvas v-for="(item, i) in showImgArr" :ref="`fingerCanvasElement_${i}`" width="1280" height="720" style="position: absolute; left: 0px; top: 0px;"></canvas> -->
       <canvas ref="fingerCanvasElement_0" width="200" height="200"></canvas>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onMounted, ref, watch } from "vue";
import {
  FilesetResolver,
  HandLandmarker,
  DrawingUtils,
} from "@mediapipe/tasks-vision";

let handLandmarker: HandLandmarker;
const runningMode = "VIDEO";

const btnText = ref("ENABLE WEBCAM");
let webcamRunning: Boolean = false;
const height = 360;
const width = 480;
const videoHeight = "360px";
const videoWidth = "480px";
const video = ref<HTMLVideoElement | null>(null);
const canvasElement = ref<HTMLCanvasElement | null>(null);
let canvasCtx: CanvasRenderingContext2D;
const fingerCanvasElement_0 = ref<HTMLCanvasElement | null>(null);
let fingerCanvasCtx_0: CanvasRenderingContext2D;
const gestureOutput = ref<HTMLElement | null>(null);
let handleEnabelWebcamButtonClick: Function;
let mediaStream = null;
const fingerArr = [4, 8, 12, 16, 20];
let showImgArr = ref<ImageData[]>([]);

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
    btnText.value = "ENABLE PREDICTIONS";
  } else {
    webcamRunning = true;
    btnText.value = "DISABLE PREDICTIONS";
  }

  // getUsermedia parameters.
  const constraints = {
    video: true,
  };

  // Activate the webcam stream.
  navigator.mediaDevices.getUserMedia(constraints).then((stream) => {
    video.value.srcObject = stream;
    mediaStream = stream;
    // video.addEventListener("loadeddata", predictWebcam);
  });
};

// 关掉视频
// const stopWebcam = () => {
//   // 删除视频流
//   if (mediaStream) {
//     video.value.srcObject = null;
//     mediaStream.getTracks().forEach((track) => track.stop());
//     mediaStream = null;
//   }

//   canvasCtx.restore();
// };

// Check if webcam access is supported.
const hasGetUserMedia = computed(() => !!navigator.mediaDevices?.getUserMedia);

onMounted(() => {
  canvasCtx = canvasElement.value?.getContext('2d');
  fingerCanvasCtx_0 = fingerCanvasElement_0.value?.getContext('2d');

  // If webcam supported, add event listener to button for when user
  // wants to activate it.
  if (hasGetUserMedia) {
    // enableWebcamButton.addEventListener("click", enableCam);
    handleEnabelWebcamButtonClick = enableCam;
  } else {
    console.warn("getUserMedia() is not supported by your browser");
  }
});

let lastVideoTime = -1;
let results = undefined;
const predictWebcam = async () => {
  // Now let's start detecting the stream.
  let nowInMs = Date.now();
  if (video.value.currentTime !== lastVideoTime) {
    lastVideoTime = video.value.currentTime;
    results = handLandmarker.detectForVideo(video.value, nowInMs);
  }

  canvasCtx.save();
  canvasCtx.clearRect(
    0,
    0,
    canvasElement.value?.width,
    canvasElement.value?.height
  );
  const drawingUtils = new DrawingUtils(canvasCtx);

  canvasElement.value.style.height = videoHeight;
  video.value.style.height = videoHeight;
  canvasElement.value.style.width = videoWidth;
  video.value.style.width = videoWidth;

  const imgArr = new Array();

  if (results.landmarks) {    
    for (let landmarks of results.landmarks) {
      landmarks = landmarks.filter((item, i) => fingerArr.includes(i));
      drawingUtils.drawLandmarks(landmarks, {
        color: "#FF0000",
        lineWidth: 2,
      });

      nextTick(() => {
        // // 获取每个手指的图片数据
        // for (const landmark of landmarks) {
        //   console.log(landmark.x * width - 5, landmark.y * height - 5, 10, 10);
        //   canvasCtx.fillStyle = 'red';
        //   canvasCtx.fillRect(50, 50, 100, 100);
        //       console.log(989898, res);
          
        //   imgArr.push(res);
        // }
      })
    }
  }
  
  showImgArr = imgArr;
  
  canvasCtx.restore();

  // Call this function again to keep predicting when the browser is ready.
  if (webcamRunning === true) {
    window.requestAnimationFrame(predictWebcam); 
    console.log(999, fingerCanvasCtx_0);
    
    // if (fingerCanvasCtx_0) {
    //   console.log(111231232, showImgArr[0]);
    //   fingerCanvasCtx_0.putImageData(showImgArr[0], 0, 0);
    // }
  }
}

</script>

<style scoped>
body {
  font-family: roboto;
  margin: 2em;
  color: #3d3d3d;
  --mdc-theme-primary: #007f8b;
  --mdc-theme-on-primary: #f1f3f4;
}

h1 {
  color: #007f8b;
}

h2 {
  clear: both;
}

video {
  clear: both;
  display: block;
  transform: rotateY(180deg);
  -webkit-transform: rotateY(180deg);
  -moz-transform: rotateY(180deg);
  height: 280px;
}

section {
  opacity: 1;
  transition: opacity 500ms ease-in-out;
}

.removed {
  display: none;
}

.invisible {
  opacity: 0.2;
}

.detectOnClick {
  position: relative;
  float: left;
  width: 48%;
  margin: 2% 1%;
  cursor: pointer;
}
.videoView {
  position: absolute;
  float: left;
  width: 48%;
  margin: 2% 1%;
  cursor: pointer;
  min-height: 500px;
}

.videoView p,
.detectOnClick p {
  padding-top: 5px;
  padding-bottom: 5px;
  background-color: #007f8b;
  color: #fff;
  border: 1px dashed rgba(255, 255, 255, 0.7);
  z-index: 2;
  margin: 0;
}

.highlighter {
  background: rgba(0, 255, 0, 0.25);
  border: 1px dashed #fff;
  z-index: 1;
  position: absolute;
}

.canvas {
  z-index: 1;
  position: absolute;
  pointer-events: none;
}

.output_canvas {
  transform: rotateY(180deg);
  -webkit-transform: rotateY(180deg);
  -moz-transform: rotateY(180deg);
}

.detectOnClick {
  z-index: 0;
  font-size: calc(8px + 1.2vw);
}

.detectOnClick img {
  width: 45vw;
}
.output {
  display: none;
  width: 100%;
  font-size: calc(8px + 1.2vw);
}
</style>
