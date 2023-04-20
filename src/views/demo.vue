<template>
  <div class="flex flex-row justify-center">
    <div class="flex flex-col">
      <image-list @changeI="changeImage($event)" />
    </div>
    <video
      id="live-video"
      v-if="isCameraStarted"
      v-show="false"
      width="640"
      height="480"
      autoplay
    />
    <canvas id="live-canvas" width="640" height="480" />
    <div class="flex flex-col">
      <div class="relative">
        <button
          class="flex items-center px-10 py-2 text-white bg-gray-800 hover:opacity-95 focus:outline-none"
        >
          Upload Image
        </button>
        <input
          type="file"
          @change="handleFileUpload"
          class="absolute top-0 left-0 w-full h-full opacity-0 cursor-pointer"
        />
      </div>
      <button
        @click="detectFace"
        type="button"
        class="px-6 py-2 text-white bg-gray-800 rounded-md hover:opacity-95 focus:outline-none"
        aria-expanded="false"
      >
        Detect Face
      </button>
      <button
        @click="extractLandmark"
        type="button"
        class="px-6 py-2 text-white bg-gray-800 rounded-md hover:opacity-95 focus:outline-none"
        aria-expanded="false"
      >
        Extract Landmark
      </button>
      <button
        @click="predictGender"
        type="button"
        class="px-6 py-2 text-white bg-gray-800 rounded-md hover:opacity-95 focus:outline-none"
        aria-expanded="false"
      >
        Estimate Gender
      </button>
      <button
        @click="predictAge"
        type="button"
        class="px-6 py-2 text-white bg-gray-800 rounded-md hover:opacity-95 focus:outline-none"
        aria-expanded="false"
      >
        Estimate Age
      </button>
    </div>
  </div>
</template>
<script>
import imageList from "./imageList.vue";
import * as faceSDK from "faceplugin";

export default {
  data() {
    return {
      mode: 0,
      emotions: {
        0: "angry",
        1: "disgust",
        2: "fear",
        3: "smile",
        4: "sad",
        5: "surprise",
        6: "neutral",
      },
      detectSession: null,
      liveSession: null,
      landmarkSession: null,
      poseSession: null,
      expressionSession: null,
      eyeSession: null,
      genderSession: null,
      ageSession: null,
      featureSession: null,
    };
  },
  components: {
    imageList,
  },
  computed: {
    isCameraStarted() {
      return this.$store.getters["isCameraStarted"];
    },
  },

  methods: {
    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = () => {
          this.selectImage(reader.result);
        };
        reader.readAsDataURL(file);
      }
    },

    async openCamera() {
      await this.$store.dispatch("startCamera").then((stream) => {
        const videoDiv = document.getElementById("live-video");
        videoDiv.srcObject = stream;
      });
      await this.displayCamera();
    },

    closeCamera() {
      this.$store.dispatch("stopCamera");
    },

    async displayCamera() {
      const video = document.getElementById("live-video");
      const canvas = document.getElementById("live-canvas");
      const canvasCtx = canvas.getContext("2d");
      canvasCtx.drawImage(video, 0, 0, 640, 480);

      switch (this.mode) {
        case 0:
          await this.detectFace();
          break;
        case 1:
          await this.extractLandmark();
          break;
        case 2:
          await this.detectLivenessDetection();
          break;
        case 3:
          await this.predictFaceExpression();
          break;
        case 4:
          await this.predictFacePose();
          break;
        case 5:
          await this.predictEyeCloseness();
          break;
        case 6:
          await this.predictGender();
          break;
        case 7:
          await this.predictAge();
          break;
        case 8:
          await this.extractFeature();
          break;
        default:
          await this.detectFace();
      }

      setTimeout(() => this.displayCamera(), 33);
    },

    changeImage(filename) {
      this.filename = filename;
      this.closeCamera();
      this.selectImage(filename);
    },

    selectImage(imageFile) {
      const canvas = document.getElementById("live-canvas");
      const canvasCtx = canvas.getContext("2d");
      canvasCtx.clearRect(0, 0, 640, 480);

      const img1 = new Image();
      img1.onload = function () {
        canvasCtx.drawImage(img1, 0, 0, 640, 480);
      };
      img1.src = imageFile;
    },

    async detectFace() {
      this.mode = 0;
      const detectionResult = await faceSDK.detectFace(
        this.detectSession,
        "live-canvas"
      );

      var bbox = detectionResult.bbox;
      var face_count = bbox.shape[0],
        bbox_size = bbox.shape[1];

      const canvas = document.getElementById("live-canvas");
      const canvasCtx = canvas.getContext("2d");
      canvasCtx.beginPath();

      for (let i = 0; i < face_count; i++) {
        var x1 = parseInt(bbox.data[i * bbox_size]),
          y1 = parseInt(bbox.data[i * bbox_size + 1]),
          x2 = parseInt(bbox.data[i * bbox_size + 2]),
          y2 = parseInt(bbox.data[i * bbox_size + 3]),
          width = Math.abs(x2 - x1),
          height = Math.abs(y2 - y1);

        canvasCtx.strokeStyle = "red";
        canvasCtx.strokeRect(x1, y1, width, height);
        canvasCtx.stroke();
      }
    },

    async extractLandmark() {
      this.mode = 1;
      const detectionResult = await faceSDK.detectFace(
        this.detectSession,
        "live-canvas"
      );
      const points = await faceSDK.predictLandmark(
        this.landmarkSession,
        "live-canvas",
        detectionResult.bbox
      );

      const canvas = document.getElementById("live-canvas");
      const canvasCtx = canvas.getContext("2d");
      canvasCtx.beginPath();

      for (let i = 0; i < points.length; i++) {
        for (let j = 0; j < 68; j++) {
          var x1 = points[i][j * 2],
            y1 = points[i][j * 2 + 1];

          canvasCtx.moveTo(x1 + 2, y1);
          canvasCtx.arc(x1, y1, 2, 0, 2 * Math.PI);
          canvasCtx.strokeStyle = "red";
          canvasCtx.stroke();
        }
      }
    },

    async predictGender() {
      this.mode = 6;
      const detectionResult = await faceSDK.detectFace(
        this.detectSession,
        "live-canvas"
      );
      const genderResult = await faceSDK.predictGender(
        this.genderSession,
        "live-canvas",
        detectionResult.bbox
      );

      const canvas = document.getElementById("live-canvas");
      const canvasCtx = canvas.getContext("2d");
      canvasCtx.beginPath();

      var face_count = genderResult.length;

      for (let i = 0; i < face_count; i++) {
        var x1 = parseInt(genderResult[i][0]),
          y1 = parseInt(genderResult[i][1]),
          x2 = parseInt(genderResult[i][2]),
          y2 = parseInt(genderResult[i][3]),
          result = genderResult[i][4] > 0.6 ? "Male" : "Female",
          width = Math.abs(x2 - x1),
          height = Math.abs(y2 - y1);

        const textWidth = canvasCtx.measureText(result).width;
        canvasCtx.fillStyle = "black";
        canvasCtx.fillRect(x1, y1 - 30, textWidth + 10, 30);
        canvasCtx.strokeStyle = "red";
        canvasCtx.fillStyle = "white";
        canvasCtx.font = "bold 20px sans-serif";
        canvasCtx.strokeRect(x1, y1, width, height);
        canvasCtx.fillText(result, x1, y1 - 10);
        canvasCtx.stroke();
      }
    },

    async predictAge() {
      this.mode = 7;
      const detectionResult = await faceSDK.detectFace(
        this.detectSession,
        "live-canvas"
      );
      const ageResult = await faceSDK.predictAge(
        this.ageSession,
        "live-canvas",
        detectionResult.bbox
      );

      const canvas = document.getElementById("live-canvas");
      const canvasCtx = canvas.getContext("2d");
      canvasCtx.beginPath();

      var face_count = ageResult.length;

      for (let i = 0; i < face_count; i++) {
        var x1 = parseInt(ageResult[i][0]),
          y1 = parseInt(ageResult[i][1]),
          x2 = parseInt(ageResult[i][2]),
          y2 = parseInt(ageResult[i][3]),
          result = parseInt(ageResult[i][4]),
          width = Math.abs(x2 - x1),
          height = Math.abs(y2 - y1);

        canvasCtx.strokeStyle = "red";
        const textWidth = canvasCtx.measureText(result).width;
        canvasCtx.fillStyle = "black";
        canvasCtx.fillRect(x1, y1 - 30, textWidth + 50, 30);
        canvasCtx.fillStyle = "white";
        canvasCtx.font = "bold 20px sans-serif";
        canvasCtx.strokeRect(x1, y1, width, height);
        canvasCtx.fillText("Age: " + result, x1, y1 - 10);
        canvasCtx.stroke();
      }
    },

    async loadModels() {
      await faceSDK.load_opencv();
      this.detectSession = await faceSDK.loadDetectionModel();
      this.landmarkSession = await faceSDK.loadLandmarkModel();
      this.genderSession = await faceSDK.loadGenderModel();
      this.ageSession = await faceSDK.loadAgeModel();
    },
  },

  mounted() {
    this.loadModels();
    this.selectImage("empty.png");
  },
};
</script>
