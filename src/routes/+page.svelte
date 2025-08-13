<script>
  import { onMount } from 'svelte';
  import * as faceapi from 'face-api.js';

  let videoEl;
  let distance = '...';
  let message = 'How is your day going ???';
  let messageSize = '';
  let videoPlaying = false;
  let detectorRunning = false;
  let stream;
  let modelsLoaded = false;

  const getCenter = (points) => {
  const sum = points.reduce((acc, p) => ({ x: acc.x + p.x, y: acc.y + p.y }), { x: 0, y: 0 });
  return {
    x: sum.x / points.length,
    y: sum.y / points.length
  };
};


  async function startCamera() {
    if (videoPlaying) return;

    // Load models once
    if (!modelsLoaded) {
      await faceapi.nets.tinyFaceDetector.loadFromUri('/models');
      await faceapi.nets.faceLandmark68Net.loadFromUri('/models');
      modelsLoaded = true;
    }

    // Start webcam stream
    stream = await navigator.mediaDevices.getUserMedia({
      video: {
        width: { ideal: 1280 },
        height: { ideal: 720 },
        aspectRatio: 16 / 9,
        facingMode: "user"
      }
    });

    videoEl.srcObject = stream;

    detectorRunning = true;
    videoPlaying = true;

    videoEl.onloadedmetadata = () => {
      detect(); // 
    };

    await videoEl.play();
  }


  function stopCamera() {
    if (stream) {
      stream.getTracks().forEach(track => track.stop());
      stream = null;
      videoPlaying = false;
      detectorRunning = false;
    }
  }

  function toggleCamera() {
    if (detectorRunning) {
      stopCamera();
    } else {
      startCamera();
    }
  }

  async function estimateDistance(result) {
    const landmarks = result.landmarks;
    const leftEye = landmarks.getLeftEye();
    const rightEye = landmarks.getRightEye();

    const left = getCenter(leftEye);
    const right = getCenter(rightEye);
    left.x /= leftEye.length;
    left.y /= leftEye.length;
    right.x /= rightEye.length;
    right.y /= rightEye.length;

    const eyeDistance = Math.hypot(right.x - left.x, right.y - left.y);
    const calibratedK = 12000;
    const mm = calibratedK / eyeDistance;
    distance = `${mm.toFixed(1)} mm`;

    if (mm < 400) {
      messageSize = 'text-lg';
    } else if (mm < 900) {
      messageSize = 'text-2xl';
    } else if (mm < 1200) {
      messageSize = 'text-4xl';
    } else {
      messageSize = 'text-6xl';
    }
  }

  async function detect() {

    console.log('Detecting...');

    const result = await faceapi
      .detectSingleFace(videoEl, new faceapi.TinyFaceDetectorOptions())
      .withFaceLandmarks();

    if (result) {
      console.log('Face detected', result);
      estimateDistance(result);
    }

    if (detectorRunning) {
      requestAnimationFrame(detect);
    }
  }


</script>


<button class="px-2 text-white bg-gray-700 rounded-2xl p-2 fixed " onclick={toggleCamera}>
  {detectorRunning ? 'Stop Camera' : 'Start Camera'}
</button>


<main class="flex-1 max-w-6xl mx-auto px-20 justify-center">
  <h1 class="text-7xl py-5 text-center font-semibold mb-5">Face Distance UI</h1>

  <h2 class="py-2 text-2xl"> Welcome to the Face-Aware Interface!</h2>
  <p class="text-lg px-4">
    This interactive application tracks your face in real time and adapts the user interface based on how far you are from your screen. No need to touch or click, just move closer or further away. As of right now there is a simple text that asks about your day but this could changed.
  </p>

  <section class="flex flex-cols-3">
    <div>
      <h3 class="py-2 mt-3 text-2xl">How is it powered?</h3>
      <ul class="list-disc px-10 text-lg">
        <li>face-api.js for facial detection and landmark recognition in the browser</li>
        <li>A Svelte-based UI that reacts dynamically to your face position</li>
        <li>Webcam input for continuous, non-intrusive distance sensing</li>
      </ul>
    </div>

    <div>
      <h4 class="py-2 mt-3 text-2xl">How does it work?</h4>
      <ul class="list-disc px-10 text-lg">
        <li>The app calculates the distance between your eyes using facial landmarks.</li>
        <li>This data is used to estimate your distance from the screen.</li>
        <li>The UI scales text or reacts based on how close or far you are.</li>
      </ul>
    </div>

    <div>
      <h5 class="py-2 mt-3 text-2xl">Why does this matter?</h5>
      <ul class="list-disc px-10 text-lg">
        <li>Reduces eye strain and encourages healthy screen habits</li>
        <li>Enables distance-aware UX in education, games, and kiosks</li>
        <li>Enables accessible interaction without needing clicks or taps</li>
      </ul>
    </div>
  </section>

  <p class="text-4xl text-center mt-10"> Start your camera and watch the interface adjust. </p>
</main>




<h1 class = "text-center font-semibold text-6xl mt-20">Face-Interface Adpater Demo</h1>
<div class="flex flex-col items-center justify-center min-h-screen text-center space-y-6">
  <div class="flex justify-center w-full">
    <video bind:this={videoEl} autoplay playsinline muted width="640" height="440"class={`rounded-2xl shadow border-2 border-gray-900 ${detectorRunning ? "opacity-100" : "opacity-0"}`}></video>
  </div>

<!-- 
  <p class="p-2 text-md bg-green-50 rounded-2xl border-2 border-green-400 font-medium">
    Estimated distance: {distance}
  </p> -->

  <h1 class={`transition-all duration-300 ease-in-out p-4 text-white bg-gray-500 border-2 border-gray-900 rounded-2xl ${messageSize} ${detectorRunning ? "opacity-100" : "opacity-0"}`}  >
    {message}
  </h1>


</div>
