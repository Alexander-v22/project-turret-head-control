<!--
=========TODO LIST============
1. change the parameters on the eruo filter to make servo transitions smoother
2. change hard dead zone to softer deadzone 
3.  


-->

<script>
  import { onMount } from 'svelte';
  import { FaceLandmarker, FilesetResolver } from '@mediapipe/tasks-vision';
  import { WS_URL } from '$lib/config.js';
  import { applyAction } from '$app/forms';



  
  let videoEl;
  let detectorRunning = false;
  let stream;
  let faceLandmarker = null;
  let ws = null;
  let lastSend = 0;

  let minCutoff = 0.8;
  let beta = 0.02;
  let d_cutoff = 0.8;

  // --- Tuning ---
  const SEND_EVERY_MS = 40;         // ~25 Hz
  const SENS_YAW = 50;
  const SENS_PITCH = 50;
  const MAX_YAW = 90;
  const MAX_PITCH = 45;
  const DEADZONE_YAW = 1.0;
  const DEADZONE_PITCH = 1.0;
  const RETRY_MS = 1000;
  const MIRROR_PREVIEW = true; //  <video> uses scaleX(-1)
 
 
// Media pie initiation 
  onMount(async () => {
    connectWS();
    window.addEventListener('beforeunload', () => ws?.close());

    const fileset = await FilesetResolver.forVisionTasks('/mediapipe'); // serve WASM assets here
    faceLandmarker = await FaceLandmarker.createFromOptions(fileset, {
      baseOptions: { modelAssetPath: '/mediapipe/face_landmarker.task' },
      runningMode: 'VIDEO',
      numFaces: 1,
      outputFaceBlendshapes: false,
      outputFacialTransformationMatrixes: true
      // Switching to 3D Head-pose matrix to try to improve YAW, pitch works good with the regular 2D detection but i think
      // since where using our eyes and nose to calculate yaw and pitch once we turn to far to one side the landmarks start to clutter
    });
  });


  // Helpers
  const clamp = (v, lo, hi) => Math.min(hi, Math.max(lo, v));
  //const dz = (v, d) => (Math.abs(v) < d ? 0 : v); going to add soft dead band instead 

  function softDeadband(v, width = 1.0) {
    const a = Math.abs(v);
    if (a <= width) return 0; // true dead zone
    const t = (a - width) / a;            // 0..1
    const s = t * t * (3 - 2 * t);        // ease
    return Math.sign(v) * s * (a - width);
  }





// =========== EURO FILTER============
function oneEuro({minCutoff= 1.0, beta = 0.02, d_cutoff = 1.0} = {}) {
  //initializng the one euro filter
  let hasPrev = false;
  let xPrev = 0;
  let dxPrev = 0;
  let tPrev = 0;
    
// r = 2pi * cutoff frequency * change in sampling freqeuncy this is the smoothing factor
  const alpha = (cutoff, dt) => {
    const r = 2 * Math.PI * cutoff * dt;
    return r / (r + 1); 
  };
  //
  const expo_smoothing = (a, x, prev) => {
  return a*x + (1 - a)* prev ;
  };

  const filter = (x, t_ms) => {
    const t = t_ms* 1e-3 //ms into seconds

    if(!hasPrev){
      hasPrev = true;
      xPrev = x;
      dxPrev = 0;
      tPrev = t;

      return x
    };

    // The filtered derivative of the signal 
    const dt = Math.max(1e-3, t - tPrev) ; // avoiding 0 or a negative number => Te​=Ti​−Ti−1​ deriv of sampling frequnecy

    //smooth deriv
    const dx = Math.max(-200, Math.min(200, (x - xPrev) / dt)); // deg/s clamp => X˙i​=(Xi​−Xi−1) Te​​
    const aD = alpha(d_cutoff,dt); // => Smoothing-Factor(Te​,fcd​​)
    const dxhat = expo_smoothing(aD, dx, dxPrev)// => exponential smoothing 

    // the filtered signal 
    const cutoff = Math.min(4.0, minCutoff + beta * Math.abs(dxhat)); // clamp to max ~4 had to change because a bad frame could cause dx to spike 
    const aX= alpha(cutoff, dt );
    const xhat = expo_smoothing(aX, x, xPrev);

    // Memorize the previous values
    xPrev = xhat;
    dxPrev = dxhat;
    tPrev = t;

    return xhat;
  };

  const setParams = (opts = {}) => {
    if(opts.minCutoff != null) minCutoff = opts.minCutoff;
    if(opts.beta != null) beta = opts.beta;
    if(opts.d_cutoff != null) d_cutoff = opts.d_cutoff;
  };
  
  const reset = () => { hasPrev = false};

  return {filter, setParams, reset};

}


  const yawFilter =  oneEuro({minCutoff, beta, d_cutoff});
  const pitchfilter = oneEuro({minCutoff, beta, d_cutoff});

$: yawFilter.setParams({ minCutoff, beta, d_cutoff });
$: pitchfilter.setParams({ minCutoff, beta, d_cutoff });

  function connectWS() {
    if (ws && (ws.readyState === WebSocket.OPEN || ws.readyState === WebSocket.CONNECTING)) return;
    ws = new WebSocket(WS_URL);
    ws.onopen = () => console.log('WS open ✅');
    ws.onmessage = (e) => console.log('ESP says:', e.data);
    ws.onerror = (e) => console.warn('WS error:', e);
    ws.onclose = () => {
      console.log('WS closed, retrying…');
      setTimeout(connectWS, RETRY_MS);
    };
  }


  function sendYawPitch(yaw, pitch) {
    const now = performance.now();
    if (!ws || ws.readyState !== WebSocket.OPEN) return;
    if (now - lastSend < SEND_EVERY_MS) return;
    lastSend = now;
    ws.send(JSON.stringify({ yaw, pitch })); // ESP expects these keys
  }
 

  async function startCamera() {
    if (detectorRunning) return;

    stream = await navigator.mediaDevices.getUserMedia({
      video: {
        width: { ideal: 640 },
        height: { ideal: 480 },
        facingMode: 'user'
      }
    });
    videoEl.srcObject = stream;

    await videoEl.play();
    detectorRunning = true;
    requestAnimationFrame(detect);   // rAF passes a timestamp ar
  }

  function stopCamera() {
    if (stream) {
      stream.getTracks().forEach((t) => t.stop());
      stream = null;
    }
   
    detectorRunning = false;

  }

  function toggleCamera() {
    detectorRunning ? stopCamera() : startCamera();
  }

  // Compute average of points
  const center = (pts) => {
    let x = 0, y = 0;
    for (const p of pts) { x += p.x; y += p.y; }
    const n = pts.length || 1;
    return { x: x / n, y: y / n };
  };

 
  function yawFromMatrixDeg(M) {
    if (!M) return null;
    // forward = 3rd column of rotation block
    let fx = M[8], fy = M[9], fz = M[10];

    // normalize to remove any scale
    const L = Math.hypot(fx, fy, fz) || 1;
    fx /= L; fy /= L; fz /= L;

    let yaw = Math.atan2(fx, fz) * 180 / Math.PI; // +right
    return yaw;
  }

    function pitchFromMatrixDeg(M) {
    if (!M) return null;
    // forward = 3rd column of rotation block
    let fx = M[8], fy = M[9], fz = M[10];

    // normalize to remove any scale
    const L = Math.hypot(fx, fy, fz) || 1;
    fx /= L; fy /= L; fz /= L;

    let pitch = Math.atan2(-fy, fz) * 180 / Math.PI; // +right
    return pitch;
  }

  function detect(ts) {
    if (!detectorRunning || !faceLandmarker || !videoEl) return;

    const res = faceLandmarker.detectForVideo(videoEl, ts);
    const M = res?.facialTransformationMatrixes?.[0]?.data;
    const lms = res?.faceLandmarks?.[0];

    // we always need 2D landmarks for pitch (and yaw fallback)
    if (lms) {

    // --- 2D features (eyes mid + nose) ---
      const leftEyeIdx  = [33,160,158,133,153,144];
      const rightEyeIdx = [362,385,387,263,373,380];
      
      const L = center(leftEyeIdx.map(i => lms[i]));
      const R = center(rightEyeIdx.map(i => lms[i]));

      
      
      const eyesMid = { x: (L.x + R.x) / 2, y: (L.y + R.y) / 2 };
      const eyesDist = Math.hypot(R.x - L.x, R.y - L.y);
      const nose = lms[1];

      if (eyesDist > 0) {
        


        const offX = (nose.x - eyesMid.x) / eyesDist;  // +right in image space
        const offY = (nose.y - eyesMid.y) / eyesDist;  // +down  in image space

        // --- YAW: 3D preferred, fallback to 2D ---
        let yawDeg = yawFromMatrixDeg(M);
        let pitchDeg = pitchFromMatrixDeg(M);

        if (yawDeg == null){
          yawDeg = offX * SENS_YAW;
        }
         if (pitchDeg == null){
          pitchDeg = offX * SENS_YAW;
        }

        // --- PITCH: always 2D (define +pitch = look UP) ---

        // --- Deadband + clamp ---
        let yaw = clamp(softDeadband(yawDeg,   DEADZONE_YAW), -MAX_YAW, +MAX_YAW);
        let pitch = clamp(softDeadband(pitchDeg, DEADZONE_PITCH), -MAX_PITCH, +MAX_PITCH);

        // --- Filters ---
        yaw = yawFilter.filter(yaw, ts);
        pitch = pitchfilter.filter(pitch, ts);

        // --- Send in ESP order ---
        sendYawPitch(yaw, pitch);

        
      }
  }
  if (detectorRunning) requestAnimationFrame(detect);
}

  
  
</script>

<button
  class="px-2 text-white bg-gray-700 rounded-2xl p-2 fixed top-4 left-2"
  on:click={toggleCamera}
>
  {detectorRunning ? 'Stop Camera' : 'Start Camera'}
</button>

<main class="flex-1 max-w-6xl mx-auto px-30 justify-center">
  <h1 class="text-7xl py-5 text-center font-semibold mb-5">Turret Face Control</h1>
  <h2 class="py-2 text-2xl ">Introduction</h2>
  <p class="text-lg px-5 mb-20">Control the turret with your head movement.</p>

<div class="grid grid-cols-3 bg-black text-white p-3 rounded-xl space-y-2 text-xl">
  
    <label class="block mr-4">minCutoff: <input type="range" min="0.1" max="5" step="0.1" bind:value={minCutoff}></label>
    <label class="block mr-4">beta: <input type="range" min="0" max="0.2" step="0.005" bind:value={beta}></label>
    <label class="block mr-4">d_cutoff: <input type="range" min="0.5" max="3" step="0.1" bind:value={d_cutoff}></label>
 
    <div class="text-xl text-white mr-4" > Minimum cutoff: {minCutoff.toFixed(2)} </div>
    <div class="text-xl text-white mr-4" > Beta: {beta.toFixed(3)}</div>
    <div class="text-xl text-white mr-4" > Constant cutoff frequnecy : {d_cutoff.toFixed(1)}</div>

    <div class="text-xl text-gray-500 mr-4" > Minimum cutoff- Smoother when when still </div>
    <div class="text-xl text-gray-500 mr-4" > Beta- More response when moving  </div>
    <div class="text-xl text-gray-500 mr-4" > Constant cutoff frequnecy- smoother (less noisy) velocity estimate {d_cutoff.toFixed(1)}</div>
</div>


</main>

<div class="flex flex-col items-center justify-center min-h-screen text-center space-y-6 ">
  <div class="flex justify-center w-full">
    <video bind:this={videoEl} autoplay playsinline muted width="640" height="440" style="transform: scaleX(-1);"
      class={`rounded-2xl shadow border-2 border-gray-900 ${detectorRunning ? 'opacity-100' : 'opacity-0'}`} ></video>
  </div>
</div>
