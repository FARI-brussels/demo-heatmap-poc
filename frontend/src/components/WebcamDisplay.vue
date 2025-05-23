<template>
  <div class="webcam-container">
    <video ref="videoPlayer" autoplay playsinline @loadedmetadata="onVideoLoaded" class="rotated-video"></video>
    <canvas ref="canvasElement" style="display: none;"></canvas>
    <p>Status: {{ status }}</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';

const emit = defineEmits(['image-captured']);

const videoPlayer = ref(null);
const canvasElement = ref(null);
const stream = ref(null);
const status = ref('Initializing...');
const isCapturing = ref(false);
let animationFrameId = null;
let lastImageData = null;
const videoLoaded = ref(false);

const onVideoLoaded = () => {
  if (videoPlayer.value && canvasElement.value) {
    canvasElement.value.width = videoPlayer.value.videoWidth;
    canvasElement.value.height = videoPlayer.value.videoHeight;
    videoLoaded.value = true;
    status.value = 'Video loaded. Ready to capture.';
    
    // If we're already capturing, start processing frames
    if (isCapturing.value) {
      processFrame();
    }
  }
};

const startWebcam = async () => {
  if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
    try {
      status.value = 'Initializing webcam...';
      const mediaStream = await navigator.mediaDevices.getUserMedia({ video: true });
      stream.value = mediaStream;
      if (videoPlayer.value) {
        videoPlayer.value.srcObject = mediaStream;
      }
      status.value = 'Webcam active, waiting for video to load...';
      isCapturing.value = true;
      
      // Only start processing frames if video has loaded
      if (videoLoaded.value) {
        processFrame();
      }
    } catch (error) {
      console.error("Error accessing webcam: ", error);
      status.value = `Error accessing webcam: ${error.message}`;
    }
  } else {
    status.value = 'getUserMedia not supported';
    console.error('getUserMedia not supported on your browser!');
  }
};

const processFrame = () => {
  if (!videoPlayer.value || !canvasElement.value || !isCapturing.value) {
    return;
  }
  
  // Check if video is actually loaded and has dimensions
  if (!videoLoaded.value || videoPlayer.value.videoWidth === 0 || videoPlayer.value.videoHeight === 0) {
    // If not ready yet, wait a bit and try again
    setTimeout(processFrame, 100);
    return;
  }
  
  const context = canvasElement.value.getContext('2d');
  if (!context) return;

  // Make sure canvas dimensions match video
  canvasElement.value.width = videoPlayer.value.videoWidth;
  canvasElement.value.height = videoPlayer.value.videoHeight;
  
  // Draw the current video frame to the canvas
  context.drawImage(videoPlayer.value, 0, 0, canvasElement.value.width, canvasElement.value.height);
  
  try {
    const currentImageData = context.getImageData(0, 0, canvasElement.value.width, canvasElement.value.height);

    const changeDetected = detectChange(currentImageData.data);

    if (changeDetected) {
      status.value = 'Significant change detected. Capturing...';
      const imageDataUrl = canvasElement.value.toDataURL('image/jpeg');
      emit('image-captured', imageDataUrl);
      lastImageData = currentImageData.data.slice(); // Update last image data after capture
      // Potentially pause or reduce frequency after a capture if needed
    } else {
      status.value = "No significant change detected.";
    }
    if (!lastImageData) { // Initialize lastImageData on first frame
      lastImageData = currentImageData.data.slice();
    }
  } catch (error) {
    console.error("Error processing frame:", error);
    status.value = `Error processing frame: ${error.message}`;
  }

  animationFrameId = requestAnimationFrame(processFrame);
};

// Placeholder for change detection
const detectChange = (currentFrameData) => {
  if (!lastImageData || lastImageData.length !== currentFrameData.length) {
    return true; // First frame or different size
  }
  let diffPixels = 0;
  const threshold = 30;
  const pixelChangeThreshold = 0.07; // 7% of pixels need to change

  for (let i = 0; i < currentFrameData.length; i += 4) {
    const r1 = lastImageData[i];
    const g1 = lastImageData[i+1];
    const b1 = lastImageData[i+2];

    const r2 = currentFrameData[i];
    const g2 = currentFrameData[i+1];
    const b2 = currentFrameData[i+2];

    const diff = Math.abs(r1 - r2) + Math.abs(g1 - g2) + Math.abs(b1 - b2);
    if (diff > threshold) {
      diffPixels++;
    }
  }
  const changedPercentage = diffPixels / (currentFrameData.length / 4);
  return changedPercentage > pixelChangeThreshold;
};

const startCapture = () => {
  // This function is kept for programmatic webcam starting
  // even though we removed the button
  if (!isCapturing.value) {
    startWebcam();
  }
};

const stopWebcam = () => {
  if (stream.value) {
    stream.value.getTracks().forEach(track => track.stop());
  }
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
  isCapturing.value = false;
  videoLoaded.value = false;
  lastImageData = null;
  status.value = 'Webcam stopped.';
};

onMounted(() => {
  // Start webcam automatically on mount
  startWebcam();
});

onBeforeUnmount(() => {
  stopWebcam();
});

</script>

<style scoped>
.webcam-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 45%; /* Adjust as needed */
}

video {
  width: 100%;
  max-width: 640px;
  height: auto;
  border: 1px solid #ccc;
}

.rotated-video {
  transform: rotate(180deg);
}

/* Removed button styling since we no longer have a button */
</style> 