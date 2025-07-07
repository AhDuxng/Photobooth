<script setup>
import { ref, onUnmounted, computed, nextTick } from 'vue';
import previewImage from '../assets/mascot-bear.png';

// --- Các ref cho trạng thái ---
const videoRef = ref(null);
const canvasRef = ref(null);
const isCameraOn = ref(false);
const isPhotoTaken = ref(false);
const photoData = ref(null);
const errorMessage = ref('');
const photosInStrip = ref([]);
const stripCaptureStep = ref(0);
const isCapturing = ref(false);
const notification = ref({ show: false, message: '' });
let notificationTimeout = null;

// --- Các ref cho cài đặt ---
const countdownTime = ref(3);
const isContinuousShot = ref(false);
const countdown = ref(0);

// --- Các ref cho giao diện chỉnh sửa ---
const activeFrameType = ref('strip');
const frameColor = ref('#FFFFFF');
const activeStickers = ref([]);

const colors = ref(['#FFFFFF', '#000000', '#FFD1DC', '#AEC6CF', '#B39EB5', '#FFB347', '#77DD77', '#FDFD96']);
const stickers = ref([
  { url: 'https://i.imgur.com/8f262kG.png', name: 'Trái tim' },
  { url: 'https://i.imgur.com/O1V4h0a.png', name: 'Nơ' },
  { url: 'https://i.imgur.com/aR30a4t.png', name: 'Mèo con' },
  { url: 'https://i.imgur.com/5V3A6G2.png', name: 'Cầu vồng' },
  { url: 'https://i.imgur.com/JzL2a1s.png', name: 'Mặt cười' },
  { url: 'https://i.imgur.com/I4y2aYt.png', name: 'Bong bóng' },
]);

// --- Các ref cho bộ lọc ---
const activeFilter = ref('filter-none');
const filters = ref([
  { name: 'Gốc', class: 'filter-none' },
  { name: 'Sắc nét', class: 'filter-contrast' },
  { name: 'Nâu đỏ', class: 'filter-sepia' },
  { name: 'Đen trắng', class: 'filter-grayscale' },
  { name: 'Cổ điển', class: 'filter-vintage' },
  { name: 'Mùa hè', class: 'filter-summer' },
]);

let stream = null;

// --- CÁC HÀM XỬ LÝ ---

const resetState = () => {
  errorMessage.value = '';
  isPhotoTaken.value = false;
  photoData.value = null;
  activeFilter.value = 'filter-none';
  photosInStrip.value = [];
  stripCaptureStep.value = 0;
  isCapturing.value = false;
  countdown.value = 0;
  activeStickers.value = [];
  frameColor.value = '#FFFFFF';
};

const startCamera = async () => {
  resetState();
  try {
    stream = await navigator.mediaDevices.getUserMedia({ video: { width: { ideal: 1280 }, height: { ideal: 720 } }, audio: false });
    if (videoRef.value) videoRef.value.srcObject = stream;
    isCameraOn.value = true;
  } catch (error) {
    errorMessage.value = "Không thể truy cập camera. Vui lòng kiểm tra lại quyền và thiết bị.";
  }
};

const stopCamera = () => {
  if (stream) stream.getTracks().forEach(track => track.stop());
  isCameraOn.value = false;
  stream = null;
};

const showNotification = (message) => {
  if (notificationTimeout) clearTimeout(notificationTimeout);
  notification.value.message = message;
  notification.value.show = true;
  notificationTimeout = setTimeout(() => { notification.value.show = false; }, 2500);
};

const captureFrame = () => {
  if (!videoRef.value || !canvasRef.value) return null;
  const video = videoRef.value;
  const canvas = canvasRef.value;
  const context = canvas.getContext('2d');
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  const videoStyle = window.getComputedStyle(video);
  context.filter = videoStyle.filter;
  context.translate(canvas.width, 0);
  context.scale(-1, 1);
  context.drawImage(video, 0, 0, canvas.width, canvas.height);
  return canvas.toDataURL('image/png');
};

const generateFinalImage = async ({ bgColor = '#FFFFFF', stickersToDraw = [] } = {}) => {
  if (photosInStrip.value.length === 0 || !canvasRef.value) return null;
  
  const canvas = canvasRef.value;
  const context = canvas.getContext('2d');
  const firstImage = new Image();
  firstImage.src = photosInStrip.value[0];
  await new Promise(resolve => firstImage.onload = resolve);

  const imgWidth = firstImage.width;
  const imgHeight = firstImage.height;

  context.filter = 'none';
  context.setTransform(1, 0, 0, 1, 0, 0);

  if (activeFrameType.value === 'single') {
    canvas.width = imgWidth;
    canvas.height = imgHeight;
    const img = new Image();
    img.src = photosInStrip.value[0];
    await new Promise(r => img.onload = r);
    context.drawImage(img, 0, 0);
  } else {
    const PADDING_X = 25;
    const PADDING_TOP = 25;
    const PADDING_BOTTOM = 25;
    const FOOTER_HEIGHT = 50;

    canvas.width = imgWidth + PADDING_X * 2;
    canvas.height = (imgHeight * 4) + PADDING_TOP + PADDING_BOTTOM + FOOTER_HEIGHT;
    
    context.fillStyle = bgColor;
    context.fillRect(0, 0, canvas.width, canvas.height);

    for (let i = 0; i < 4; i++) {
      if (!photosInStrip.value[i]) continue;
      const img = new Image();
      img.src = photosInStrip.value[i];
      await new Promise(r => img.onload = r);
      const yPos = PADDING_TOP + (i * imgHeight);
      context.drawImage(img, PADDING_X, yPos, imgWidth, imgHeight);
    }
    
    context.fillStyle = '#374151'; // text-gray-700
    context.font = '22px "Be Vietnam Pro"';
    context.textAlign = 'center';
    const footerTextY = canvas.height - PADDING_BOTTOM;
    context.fillText('DEMO STUDIO', canvas.width / 2, footerTextY);
  }

  for (const sticker of stickersToDraw) {
      const stickerImg = new Image();
      stickerImg.crossOrigin = "Anonymous";
      stickerImg.src = sticker.url;
      await new Promise(r => stickerImg.onload = r);
      context.drawImage(stickerImg, sticker.x * canvas.width - (sticker.size/2), sticker.y * canvas.height - (sticker.size/2), sticker.size, sticker.size);
  }

  return canvas.toDataURL('image/png');
};


const runCountdown = (seconds) => {
  return new Promise(resolve => {
    countdown.value = seconds;
    const interval = setInterval(() => {
      countdown.value--;
      if (countdown.value <= 0) {
        clearInterval(interval);
        resolve();
      }
    }, 1000);
  });
};

const takePhoto = async () => {
  if (isCapturing.value) return;
  isCapturing.value = true;

  if (activeFrameType.value === 'strip' && isContinuousShot.value) {
    for (let i = 0; i < 4; i++) {
      await runCountdown(countdownTime.value);
      const capturedPhoto = captureFrame();
      if (capturedPhoto) {
        photosInStrip.value.push(capturedPhoto);
        stripCaptureStep.value++;
        showNotification(`Đã chụp ảnh ${stripCaptureStep.value}/4!`);
      }
      if (i < 3) await new Promise(resolve => setTimeout(resolve, 1500));
    }
  } else {
    await runCountdown(countdownTime.value);
    const capturedPhoto = captureFrame();
    if (!capturedPhoto) {
      isCapturing.value = false;
      return;
    }
    photosInStrip.value.push(capturedPhoto);
    if (activeFrameType.value === 'single') {
        // không cần tăng stripCaptureStep cho ảnh đơn
    } else {
      stripCaptureStep.value++;
    }
  }

  if (stripCaptureStep.value >= 4 || activeFrameType.value === 'single') {
    showNotification('Đang tạo ảnh của bạn...');
    photoData.value = await generateFinalImage();
    isPhotoTaken.value = true;
    stopCamera();
  } else {
    isCapturing.value = false;
  }
};

const retakePhoto = () => {
  startCamera();
};

const addSticker = (sticker) => {
  const newSticker = {
    ...sticker,
    id: Date.now(),
    x: 0.1 + Math.random() * 0.8,
    y: 0.1 + Math.random() * 0.8,
    size: 80,
  };
  activeStickers.value.push(newSticker);
};

const downloadFinalImage = async () => {
    const finalImageData = await generateFinalImage({
        bgColor: frameColor.value,
        stickersToDraw: activeStickers.value
    });
    if (finalImageData) {
        const link = document.createElement('a');
        link.download = `photobooth-final.png`;
        link.href = finalImageData;
        link.click();
    }
};

const captureButtonText = computed(() => {
  if (activeFrameType.value === 'strip' && isCameraOn.value && !isPhotoTaken) {
    return `Chụp ảnh (${stripCaptureStep.value + 1}/4)`;
  }
  return 'Chụp ảnh';
});

onUnmounted(() => {
  stopCamera();
});
</script>

<template>
  <div class="p-4 md:p-8">
    
    <div v-if="!isPhotoTaken" class="w-full max-w-6xl mx-auto bg-white/60 backdrop-blur-lg rounded-2xl shadow-xl p-6 md:p-8">
      <h1 class="text-3xl font-bold text-sky-800 mb-6 font-poppins text-center flex items-center justify-center gap-2">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor" class="w-8 h-8 text-pink-500"><path fill-rule="evenodd" d="M3 6a3 3 0 013-3h12a3 3 0 013 3v12a3 3 0 01-3 3H6a3 3 0 01-3-3V6zm4.5 9a1.5 1.5 0 100-3 1.5 1.5 0 000 3z" clip-rule="evenodd" /><path d="M12.75 12a1.5 1.5 0 11-3 0 1.5 1.5 0 013 0z" /><path fill-rule="evenodd" d="M16.5 13.5a1.5 1.5 0 100-3 1.5 1.5 0 000 3zM4.5 8.25a.75.75 0 01.75-.75h3a.75.75 0 010 1.5h-3a.75.75 0 01-.75-.75zM4.5 12a.75.75 0 01.75-.75h3a.75.75 0 010 1.5h-3a.75.75 0 01-.75-.75z" clip-rule="evenodd" /></svg>
        Photobooth Pro
      </h1>
      <div class="flex flex-col md:flex-row gap-8">
        <div class="w-full md:w-1/4 lg:w-1/5">
          <h3 class="text-base font-semibold text-sky-800 mb-3">Loại khung</h3>
          <div class="space-y-4">
            <div @click="activeFrameType = 'single'" class="cursor-pointer group">
              <div class="w-28 mx-auto">
                <div class="bg-white p-1.5 rounded-lg shadow-sm border-2 transition-all" :class="[activeFrameType === 'single' ? 'border-sky-500 ring-2 ring-sky-300' : 'border-gray-200 hover:border-sky-400']">
                  <div class="w-full h-32 bg-gray-200 rounded-sm flex items-center justify-center overflow-hidden"><img v-if="photosInStrip[0]" :src="photosInStrip[0]" class="w-full h-full object-cover"></div>
                </div>
              </div>
              <p class="text-center mt-2 text-sm font-medium" :class="[activeFrameType === 'single' ? 'text-sky-600' : 'text-gray-600']">Ảnh đơn</p>
            </div>
            <div @click="activeFrameType = 'strip'" class="cursor-pointer group">
              <div class="w-28 mx-auto">
                <div class="bg-white rounded-lg shadow-sm border-2 transition-all overflow-hidden" :class="[activeFrameType === 'strip' ? 'border-sky-500 ring-2 ring-sky-300' : 'border-gray-200 hover:border-sky-400']">
                  <div class="p-1.5 space-y-1.5">
                    <div v-for="i in 4" :key="i" class="h-12 bg-gray-200" :class="{'ring-2 ring-pink-500 ring-inset': activeFrameType === 'strip' && stripCaptureStep === i - 1 && isCameraOn}"><img v-if="photosInStrip[i-1]" :src="photosInStrip[i-1]" class="w-full h-full object-cover"></div>
                  </div>
                </div>
              </div>
              <p class="text-center mt-2 text-sm font-medium" :class="[activeFrameType === 'strip' ? 'text-sky-600' : 'text-gray-600']">Dải 4 ảnh</p>
            </div>
          </div>
        </div>

        <div class="w-full md:w-3/4 lg:w-4/5">
            <div class="relative w-full aspect-video bg-gray-900 rounded-lg overflow-hidden flex items-center justify-center mb-4">
              <div v-if="!isCameraOn" class="text-center text-gray-400"><p>Camera đang tắt</p></div>
              <video ref="videoRef" v-show="isCameraOn" :class="activeFilter" autoplay playsinline muted class="w-full h-full object-cover"></video>
              <div v-if="countdown > 0" class="absolute inset-0 bg-black/50 flex items-center justify-center text-white text-9xl font-bold z-20">{{ countdown }}</div>
              <transition name="fade"><div v-if="notification.show" class="absolute bottom-4 left-1/2 -translate-x-1/2 bg-black/60 text-white text-sm px-4 py-2 rounded-full z-20">{{ notification.message }}</div></transition>
              <canvas ref="canvasRef" class="hidden"></canvas>
            </div>
            <div v-if="isCameraOn" class="mb-4">
              <div class="flex space-x-4 overflow-x-auto pb-3">
                <div v-for="filter in filters" :key="filter.class" @click="activeFilter = filter.class" class="flex-shrink-0 cursor-pointer text-center group">
                  <div class="w-16 h-16 rounded-lg overflow-hidden border-2" :class="[activeFilter === filter.class ? 'border-sky-500 ring-2 ring-sky-300' : 'border-transparent']"><img :src="previewImage" :class="filter.class" class="w-full h-full object-cover" alt="Preview"></div>
                  <p class="mt-1.5 text-xs font-semibold" :class="[activeFilter === filter.class ? 'text-sky-600' : 'text-gray-600']">{{ filter.name }}</p>
                </div>
              </div>
            </div>
            <div class="flex flex-col sm:flex-row justify-center items-center gap-4 border-t pt-4 mt-auto">
                <div class="flex flex-col items-center">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Hẹn giờ</label>
                    <div class="flex justify-center gap-2">
                      <button v-for="time in [3, 5, 10]" :key="time" @click="countdownTime = time" class="w-10 h-10 text-sm rounded-full border-2 transition" :class="[countdownTime === time ? 'bg-sky-500 text-white border-sky-500' : 'bg-white text-sky-700 border-sky-300']">{{ time }}s</button>
                    </div>
                </div>
                <div class="flex-grow">
                    <button v-if="!isCameraOn" @click="startCamera" class="w-full px-8 py-3 bg-sky-500 text-white font-semibold rounded-full">Bật Camera</button>
                    <button v-else :disabled="isCapturing" @click="takePhoto" class="w-full h-full text-lg inline-flex items-center justify-center px-6 py-3 bg-sky-500 text-white font-semibold rounded-full disabled:bg-gray-400">
                      <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 9a2 2 0 012-2h.93a2 2 0 001.664-.89l.812-1.22A2 2 0 0110.07 4h3.86a2 2 0 011.664.89l.812 1.22A2 2 0 0018.07 7H19a2 2 0 012 2v9a2 2 0 01-2 2H5a2 2 0 01-2-2V9z" /><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 13a3 3 0 11-6 0 3 3 0 016 0z" /></svg>
                      <span>{{ captureButtonText }}</span>
                    </button>
                </div>
                <div v-if="activeFrameType === 'strip' && isCameraOn" class="flex flex-col items-center">
                    <label class="block text-sm font-medium text-gray-700 mb-1">Liên tục</label>
                    <button @click="isContinuousShot = !isContinuousShot" class="relative inline-flex items-center h-6 rounded-full w-11 transition-colors" :class="[isContinuousShot ? 'bg-sky-500' : 'bg-gray-300']">
                      <span class="inline-block w-4 h-4 transform bg-white rounded-full transition-transform" :class="[isContinuousShot ? 'translate-x-6' : 'translate-x-1']"></span>
                    </button>
                </div>
            </div>
        </div>
      </div>
    </div>

    <div v-else class="w-full max-w-5xl flex flex-col md:flex-row gap-8">
        <div class="w-full md:w-1/2 flex flex-col items-center">
            <div class="relative w-full max-w-sm">
                <img :src="photoData" alt="Ảnh đã chụp" class="w-full h-auto rounded-lg shadow-lg bg-white">
                <div v-for="sticker in activeStickers" :key="sticker.id" class="absolute" :style="{ left: `${sticker.x * 100}%`, top: `${sticker.y * 100}%`, transform: 'translate(-50%, -50%)' }">
                    <img :src="sticker.url" :style="{ width: `${sticker.size}px` }" alt="sticker">
                </div>
            </div>
        </div>
        <div class="w-full md:w-1/2">
            <div class="space-y-6">
                <div v-if="activeFrameType === 'strip'">
                    <h4 class="font-semibold text-gray-800 mb-2">Màu khung</h4>
                    <div class="flex flex-wrap gap-2">
                        <button v-for="color in colors" :key="color" @click="frameColor = color" class="w-8 h-8 rounded-full border-2" :style="{ backgroundColor: color }" :class="[frameColor === color ? 'ring-2 ring-offset-2 ring-sky-500' : 'border-gray-200']"></button>
                    </div>
                </div>
                <div>
                    <h4 class="font-semibold text-gray-800 mb-2">Sticker</h4>
                    <div class="grid grid-cols-4 sm:grid-cols-6 gap-3 p-3 bg-gray-100 rounded-lg">
                        <button v-for="sticker in stickers" :key="sticker.name" @click="addSticker(sticker)" class="aspect-square bg-white rounded-lg flex items-center justify-center p-1 hover:bg-sky-100 transition">
                            <img :src="sticker.url" :alt="sticker.name" class="max-w-full max-h-full">
                        </button>
                    </div>
                </div>
                <div class="flex flex-col sm:flex-row gap-4 pt-4 border-t">
                    <button @click="retakePhoto" class="w-full px-6 py-3 bg-gray-200 text-gray-800 font-semibold rounded-full hover:bg-gray-300">Chụp lại</button>
                    <button @click="downloadFinalImage" class="w-full px-6 py-3 bg-green-500 text-white font-semibold rounded-full hover:bg-green-600">Tải xuống</button>
                </div>
            </div>
        </div>
    </div>
  </div>
</template>

<style scoped>
/* ... (giữ nguyên toàn bộ style) ... */
video { transform: scaleX(-1); }
.filter-none { filter: none; }
.filter-grayscale { filter: grayscale(100%); }
.filter-sepia { filter: sepia(100%); }
.filter-contrast { filter: contrast(140%); }
.filter-vintage { filter: sepia(65%) contrast(110%) brightness(90%) saturate(130%); }
.filter-summer { filter: contrast(110%) brightness(110%) saturate(150%) hue-rotate(-10deg); }
.overflow-x-auto::-webkit-scrollbar { height: 6px; }
.overflow-x-auto::-webkit-scrollbar-thumb { background-color: #94a3b8; border-radius: 10px; }
.fade-enter-active, .fade-leave-active { transition: opacity 0.5s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>