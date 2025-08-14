<!--
1. create the code for the yaw and pitch 
2. create the ESP32 "Transfer"
3. make sure it works 


-->

<script>
  import { onMount } from 'svelte';
  import * as faceapi from 'face-api.js';

  let videoEl;
  let videoPlaying = false;
  let detectorRunning = false;
  let stream;
  let modelsLoaded = false;

  // CERATING VARIABLES FOR TURRET
  const SENS_YAW = 70;
  const SENS_PITCH = 70;
  const MAX_YAW = 90;
  const MAX_PITCH = 45; 

  // Gets the center of your face for accuarcy
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

  // the detection loop for tracking face and calculating yaw/pitch
  async function detect () {

    console.log('Detecting...');

    const result = await faceapi
      .detectSingleFace(videoEl, new faceapi.TinyFaceDetectorOptions())
      .withFaceLandmarks();

    if (result) {
      console.log('Face detected', result);

      // DECLARING OUR FACIAL FEATURES TO CONTORL YAW AND PITCH
      // The reason why these are crucial is due to the way the landmark systems
      // since these are image points and not real world coordniates you need to "map" them accordingly
      const lm = result.landmarks;
      const leftEye = lm.getLeftEye();
      const rightEye = lm.getRightEye();
      const nose = lm.getNose();
      const noseTip = nose[nose.length-1];

      // Need the cneter of our eyes for precision      
      // Need to find the middle points of the eye and the distance between the two
      const L = getCenter(leftEye);
      const R = getCenter(rightEye); // fixed function call typo
      const eyesMid = {x: (L.x + R.x)/2, y: (L.y + R.y) / 2};
      const eyesDist = Math.hypot(R.x - L.x ,R.y - L.y); // distance betweem the left and right eyes  (need since were doing 2D mapping)

      // Guard make sures the system wont collapse
      // notes: (noseTip.x - eyesMid.x) this is vector from the head center add we divide by eyeDIst to make the offset scale-invariant
      if (eyesDist > 0) {
        const offX = (noseTip.x - eyesMid.x) / eyesDist; 
        const offY = (noseTip.y - eyesMid.y) / eyesDist;

        // Mapping offsets to the angles 
        let yaw = offX * SENS_YAW;
        let pitch = -offY * SENS_PITCH ;

        console.log("Yaw before clamp", yaw.toFixed(3));
        console.log("pitch before clamp", pitch.toFixed(3));

        //CLAMPS THE TARGETS FROM ABOVE to prevent any issues with the hardware
        yaw =  Math.max(-MAX_YAW, Math.min(MAX_YAW, yaw));
        pitch = Math.max(-MAX_PITCH, Math.min(MAX_PITCH, pitch)); 
        console.log("Yaw after clamp", yaw.toFixed(3))
        console.log("Pitch after clamp", pitch.toFixed(3));

        // need to send to the esp32 now 
   
      }
    }

    if (detectorRunning) {
      requestAnimationFrame(detect);
    }
  }
</script>



<button class="px-2 text-white bg-gray-700 rounded-2xl p-2 fixed top-4 left-2" onclick={toggleCamera}>
  {detectorRunning ? 'Stop Camera' : 'Start Camera'}
</button>


<main class="flex-1 max-w-6xl mx-auto px-20 justify-center">
  <h1 class="text-7xl py-5 text-center font-semibold mb-5">Turret Face Control</h1>

  <h2 class="py-2 text-2xl px-15"> Introduction</h2>
  <p class="text-lg px-20">
    Page to be able to control a turret with your head movement
  </p>

  
</main>


<div class="flex flex-col items-center justify-center min-h-screen text-center space-y-6">
  <div class="flex justify-center w-full">
    <video bind:this={videoEl} autoplay playsinline muted width="640" height="440"class={`rounded-2xl shadow border-2 border-gray-900 ${detectorRunning ? "opacity-100" : "opacity-0"}`}></video>
  </div>

  


</div>
