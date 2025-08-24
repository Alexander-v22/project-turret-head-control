# Turret Face Control — Frontend README

![C](https://img.shields.io/badge/Language-C-blue)
![ESP-IDF](https://img.shields.io/badge/Framework-ESP--IDF-green)
![FreeRTOS](https://img.shields.io/badge/RTOS-FreeRTOS-orange)
![WiFi](https://img.shields.io/badge/Connectivity-WiFi-lightgrey)
![WebSocket](https://img.shields.io/badge/Protocol-WebSocket-purple)


This page is the **browser side** of a two-part system that lets you control a **pan/tilt turret** with your **head movement**. It demonstrates **real-time computer vision**, **signal processing**, and a clean **human-to-robot control loop** that streams directly to embedded hardware.

---

## What This Page Does

- Captures your webcam feed and runs **MediaPipe Face Landmarker** in the browser (WASM).
- Measures the **nose tip** relative to the **eye center** to infer head **yaw** (left/right) and **pitch** (up/down).
- Normalizes by eye-to-eye distance (works at different camera distances).
- Converts offsets to **angles in degrees**, smooths them, and sends them over **WebSocket** to the turret.

> In short: look around → page computes angles → hardware follows.

---

## How Motion Is Derived (Conceptual)

1. **Landmarks → Features**  
   - Average clusters around left/right eyes → **eyesMid**  
   - Use **nose tip** and **eyesMid** to get normalized offsets `(offX, offY)`.

2. **Normalization**  
   - Divide by **eyes distance** so scale stays consistent.

3. **Angles**  
   - `yaw   =  offX * SENS_YAW` → clamped to `[-90°, +90°]`  
   - `pitch = -offY * SENS_PITCH` → clamped to `[-45°, +45°]`  
   - (Negative pitch flips camera coordinates into “look up is +”.)

4. **Smoothing**  
   - **Soft deadband**: suppress micro-jitter near center without a hard snap.  
   - **One Euro filter**: adaptive smoothing that’s steady when still and responsive when fast.

5. **Streaming**  
   - Sends at approximately **25 Hz** (`SEND_EVERY_MS = 40`) to the ESP32 over WebSocket as JSON.

---

## Signal Processing

**One Euro Filter**  
- `minCutoff` — baseline smoothness when still (lower = smoother, higher = more responsive).  
- `beta` — increases responsiveness proportional to motion speed (move faster → more bandwidth).  
- `d_cutoff` — smoothing for the velocity estimate (reduces noisy spikes).

**Live Tuning UI**  
- Sliders for `minCutoff`, `beta`, and `d_cutoff` allow real-time adjustment to find the best balance between responsiveness and stability.

---

## Data Contract (WebSocket)

- **Endpoint**: `ws://<ESP32-IP>:81/ws`  
- **Rate**: ~25 Hz  
- **Payload**:

```json
{ "yaw": <float degrees>, "pitch": <float degrees> }
```

## Expected Ranges from the Page

- **Yaw**: `[-90°, +90°]`  
- **Pitch**: `[-45°, +45°]`  

The firmware parses these values and maps them to servo angles.



## Hardware Link

On the hardware side, an **ESP32 microcontroller** receives these messages and drives the physical turret.

- Maps `yaw/pitch` into **servo angles (0–180°)**.  
- **Pan servo** (left/right) and **tilt servo** (up/down) follow your head in real time.  
- If the WebSocket connection drops, the ESP32 automatically switches to a **joystick fallback** so the turret remains usable.  
- **Mode switching** is handled by buttons and LEDs on the ESP32 board:  
  - **Red LED** → Joystick mode  
  - **Blue LED** → Head control mode  

Additional features include LED feedback wiring for **blink/joystick click** events, with room for expansion (such as an LCD mode display).

---


## End-to-End Flow

1. Webcam → Face Tracking → Offset to Angles  
2. → Soft Deadband + One Euro Filter  
3. → JSON `{yaw, pitch}` @ 25Hz  
4. → WebSocket → ESP32  
5. → Servo PWM → Turret moves  

