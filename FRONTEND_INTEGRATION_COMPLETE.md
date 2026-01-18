# TRACK D - Frontend Integration Guide

**Quick Start for Frontend Developers**

This guide contains everything you need to integrate TRACK D fingerprint liveness detection into your web application.

---

## 🎯 Server Information

**WebSocket Server URL:**
```
ws://34.61.165.1:8765
```

**Status:** ✅ Live and Running
**Location:** Google Cloud Platform (GCP)
**Uptime:** 24/7 with auto-restart

---

## 📋 Table of Contents

1. [Quick Start](#quick-start)
2. [How It Works](#how-it-works)
3. [WebSocket API](#websocket-api)
4. [Complete Integration Example](#complete-integration-example)
5. [Message Formats](#message-formats)
6. [React Example](#react-example)
7. [Error Handling](#error-handling)
8. [Testing](#testing)
9. [Production Checklist](#production-checklist)

---

## 🚀 Quick Start

### Minimal Example (30 seconds to test)

```html
<!DOCTYPE html>
<html>
<head>
  <title>TRACK D Test</title>
</head>
<body>
  <h1>TRACK D Liveness Detection</h1>
  <video id="video" width="640" height="480" autoplay></video>
  <div id="status">Connecting...</div>
  <div id="result"></div>

  <script>
    // 1. Connect to WebSocket
    const ws = new WebSocket('ws://34.61.165.1:8765');
    const video = document.getElementById('video');
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');

    ws.onopen = async () => {
      document.getElementById('status').textContent = 'Connected! Starting camera...';

      // 2. Get camera access
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { width: 800, height: 600 }
      });
      video.srcObject = stream;

      // 3. Start analysis
      ws.send(JSON.stringify({ command: 'START_ANALYSIS' }));

      // 4. Send frames (10 FPS)
      setInterval(() => {
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        ctx.drawImage(video, 0, 0);
        const frameData = canvas.toDataURL('image/jpeg', 0.8);
        ws.send(JSON.stringify({ type: 'frame', frame: frameData }));
      }, 100);
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      document.getElementById('status').textContent = `Status: ${data.status}`;

      if (data.result) {
        document.getElementById('result').innerHTML =
          `<h2>${data.result} - ${data.confidence}%</h2>`;
      }
    };
  </script>
</body>
</html>
```

**Save as `test.html` and open in browser!**

---

## 🔄 How It Works

### Architecture Flow

```
┌─────────────────────────────────────────────────────────────┐
│ BROWSER (Your Frontend)                                      │
├─────────────────────────────────────────────────────────────┤
│ 1. getUserMedia() → Capture camera                          │
│ 2. Draw video frame to canvas                               │
│ 3. Convert to base64 JPEG                                   │
│ 4. Send via WebSocket → ws://34.61.165.1:8765              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ SERVER (GCP - 34.61.165.1:8765)                             │
├─────────────────────────────────────────────────────────────┤
│ 1. Receive frame (base64 JPEG)                              │
│ 2. Decode to OpenCV image                                   │
│ 3. Detect finger (MediaPipe)                                │
│ 4. Analyze liveness (6 detection methods)                   │
│ 5. Calculate scores & detect attacks                        │
│ 6. Send results back ↓                                      │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ BROWSER (Your Frontend)                                      │
├─────────────────────────────────────────────────────────────┤
│ 1. Receive results via WebSocket                            │
│ 2. Display status (WAITING/ANALYZING/LIVE/SPOOF)           │
│ 3. Show confidence score                                    │
│ 4. Handle result (allow/deny access)                        │
└─────────────────────────────────────────────────────────────┘
```

### Detection Timeline

```
0s  → User opens page
1s  → Camera permission granted
2s  → WebSocket connected, START_ANALYSIS sent
3s  → User places finger → Status: WAITING
4s  → Finger detected → Status: ANALYZING
9s  → Analysis complete → Status: LIVE or SPOOF (result displayed)
12s → Auto-restart → Ready for next person
```

---

## 📡 WebSocket API

### Connection

```javascript
const ws = new WebSocket('ws://34.61.165.1:8765');
```

### Commands (Frontend → Server)

| Command | Description | When to Use |
|---------|-------------|-------------|
| `START_ANALYSIS` | Begin verification session | User clicks "Start Scan" |
| `RESET` | Clear current analysis | User wants to try again |
| `SAVE_RESULT` | Save verification result | User wants to save (LIVE only) |
| `STOP_ANALYSIS` | End session | User cancels |

**Format:**
```javascript
ws.send(JSON.stringify({ command: "START_ANALYSIS" }));
```

### Frame Data (Frontend → Server)

**Format:**
```javascript
ws.send(JSON.stringify({
  type: "frame",
  frame: "data:image/jpeg;base64,/9j/4AAQSkZJRgABA..."
}));
```

**Recommended Settings:**
- **Frame Rate:** 10 FPS (100ms interval)
- **Image Format:** JPEG
- **Quality:** 0.8 (80%)
- **Resolution:** 640x480 or 800x600

---

## 📨 Message Formats

### Server → Frontend Messages

#### 1. Connection Message (on connect)

```json
{
  "type": "connection",
  "message": "Connected to TRACK D Cloud Server",
  "status": "ready",
  "version": "cloud-1.0",
  "commands": ["START_ANALYSIS", "RESET", "SAVE_RESULT", "STOP_ANALYSIS"],
  "note": "Send camera frames with type='frame'"
}
```

#### 2. Detection Data (real-time, 10 times/second)

```json
{
  "timestamp": "2026-01-18T12:47:30.123456",
  "frame_count": 45,
  "status": "ANALYZING",
  "finger_detected": true,
  "scores": {
    "motion": 85.5,
    "texture": 72.3,
    "edge_density": 68.1,
    "color_variance": 81.2,
    "pattern_detection": 65.4,
    "consistency": 75.0,
    "overall": 74.5
  },
  "result": null,
  "attack_type": null,
  "confidence": 74.5,
  "ui_elements": {
    "instruction": "Analyzing liveness...",
    "progress": 60.0
  },
  "frames_analyzed": 12
}
```

#### 3. Final Result

```json
{
  "timestamp": "2026-01-18T12:47:35.123456",
  "frame_count": 150,
  "status": "LIVE",
  "finger_detected": true,
  "scores": {
    "motion": 92.1,
    "texture": 78.5,
    "edge_density": 71.3,
    "color_variance": 88.7,
    "pattern_detection": 69.8,
    "consistency": 82.4,
    "overall": 82.3
  },
  "result": "LIVE",
  "attack_type": null,
  "confidence": 82.3,
  "ui_elements": {
    "instruction": "Verification successful!",
    "progress": 100.0
  },
  "frames_analyzed": 35
}
```

#### 4. Spoof Detected

```json
{
  "status": "SPOOF",
  "result": "SPOOF",
  "attack_type": "screen_attack",
  "confidence": 25.3,
  "ui_elements": {
    "instruction": "Screen detected - Use real finger"
  }
}
```

### Status Values

| Status | Meaning | UI Suggestion |
|--------|---------|---------------|
| `WAITING` | No finger detected | "Place your finger in front of camera" |
| `ANALYZING` | Processing finger | Show progress bar, "Analyzing..." |
| `LIVE` | Real finger verified ✓ | Green checkmark, "Verification successful" |
| `SPOOF` | Attack detected ✗ | Red X, "Spoof detected" + attack type |

### Attack Types

| Attack Type | Description | User Message |
|-------------|-------------|--------------|
| `photo_attack` | Printed photo/fingerprint | "Photo detected - Use real finger" |
| `screen_attack` | Phone/tablet screen | "Screen detected - Use real finger" |
| `video_replay` | Video replay attack | "Video detected - Use real finger" |
| `fake_finger` | Silicone/rubber replica | "Fake finger detected - Use real finger" |
| `null` | No attack (LIVE result) | - |

---

## 💻 Complete Integration Example

### HTML + JavaScript (Vanilla)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>TRACK D - Fingerprint Liveness Detection</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 20px;
    }

    .container {
      background: white;
      border-radius: 20px;
      box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
      padding: 40px;
      max-width: 900px;
      width: 100%;
    }

    h1 {
      text-align: center;
      color: #333;
      margin-bottom: 30px;
      font-size: 32px;
    }

    #video {
      width: 100%;
      max-width: 640px;
      height: auto;
      border-radius: 10px;
      border: 3px solid #ddd;
      display: block;
      margin: 0 auto 20px;
    }

    .status-bar {
      background: #f5f5f5;
      padding: 20px;
      border-radius: 10px;
      margin-bottom: 20px;
    }

    .status {
      font-size: 24px;
      font-weight: bold;
      text-align: center;
      margin-bottom: 10px;
    }

    .status.waiting { color: #999; }
    .status.analyzing { color: #3b82f6; }
    .status.live { color: #10b981; }
    .status.spoof { color: #ef4444; }

    .instruction {
      text-align: center;
      font-size: 16px;
      color: #666;
      margin-bottom: 15px;
    }

    .progress-bar {
      width: 100%;
      height: 30px;
      background: #e5e7eb;
      border-radius: 15px;
      overflow: hidden;
      margin-bottom: 20px;
    }

    .progress-fill {
      height: 100%;
      background: linear-gradient(90deg, #3b82f6, #8b5cf6);
      transition: width 0.3s ease;
      display: flex;
      align-items: center;
      justify-content: center;
      color: white;
      font-weight: bold;
    }

    .scores {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
      gap: 15px;
      margin-bottom: 20px;
    }

    .score-card {
      background: #f9fafb;
      padding: 15px;
      border-radius: 8px;
      text-align: center;
    }

    .score-label {
      font-size: 12px;
      color: #6b7280;
      text-transform: uppercase;
      margin-bottom: 5px;
    }

    .score-value {
      font-size: 24px;
      font-weight: bold;
      color: #3b82f6;
    }

    .controls {
      display: flex;
      gap: 10px;
      justify-content: center;
      flex-wrap: wrap;
    }

    button {
      padding: 12px 30px;
      font-size: 16px;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      font-weight: 600;
      transition: all 0.3s;
    }

    .btn-primary {
      background: #3b82f6;
      color: white;
    }

    .btn-primary:hover {
      background: #2563eb;
      transform: translateY(-2px);
    }

    .btn-secondary {
      background: #6b7280;
      color: white;
    }

    .btn-secondary:hover {
      background: #4b5563;
    }

    .result-card {
      background: linear-gradient(135deg, #10b981, #059669);
      color: white;
      padding: 30px;
      border-radius: 15px;
      text-align: center;
      margin-top: 20px;
      display: none;
    }

    .result-card.spoof {
      background: linear-gradient(135deg, #ef4444, #dc2626);
    }

    .result-card h2 {
      font-size: 36px;
      margin-bottom: 10px;
    }

    .result-card p {
      font-size: 18px;
      opacity: 0.9;
    }

    .connection-status {
      text-align: center;
      padding: 10px;
      border-radius: 8px;
      margin-bottom: 20px;
      font-weight: 600;
    }

    .connection-status.connected {
      background: #d1fae5;
      color: #065f46;
    }

    .connection-status.disconnected {
      background: #fee2e2;
      color: #991b1b;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>🔐 Fingerprint Liveness Detection</h1>

    <div id="connectionStatus" class="connection-status disconnected">
      Disconnected
    </div>

    <video id="video" autoplay playsinline></video>

    <div class="status-bar">
      <div id="status" class="status waiting">DISCONNECTED</div>
      <div id="instruction" class="instruction">Click "Start Verification" to begin</div>

      <div class="progress-bar">
        <div id="progress" class="progress-fill" style="width: 0%">
          <span id="progressText">0%</span>
        </div>
      </div>
    </div>

    <div class="scores">
      <div class="score-card">
        <div class="score-label">Motion</div>
        <div id="score-motion" class="score-value">0%</div>
      </div>
      <div class="score-card">
        <div class="score-label">Texture</div>
        <div id="score-texture" class="score-value">0%</div>
      </div>
      <div class="score-card">
        <div class="score-label">Edge</div>
        <div id="score-edge" class="score-value">0%</div>
      </div>
      <div class="score-card">
        <div class="score-label">Color</div>
        <div id="score-color" class="score-value">0%</div>
      </div>
      <div class="score-card">
        <div class="score-label">Consistency</div>
        <div id="score-consistency" class="score-value">0%</div>
      </div>
      <div class="score-card">
        <div class="score-label">Overall</div>
        <div id="score-overall" class="score-value">0%</div>
      </div>
    </div>

    <div id="resultCard" class="result-card">
      <h2 id="resultTitle"></h2>
      <p id="resultMessage"></p>
    </div>

    <div class="controls">
      <button id="startBtn" class="btn-primary">Start Verification</button>
      <button id="resetBtn" class="btn-secondary">Reset</button>
      <button id="saveBtn" class="btn-secondary">Save Result</button>
    </div>
  </div>

  <script>
    // Configuration
    const SERVER_URL = 'ws://34.61.165.1:8765';
    const FRAME_RATE = 10; // Send 10 frames per second

    // Elements
    const video = document.getElementById('video');
    const statusEl = document.getElementById('status');
    const instructionEl = document.getElementById('instruction');
    const progressEl = document.getElementById('progress');
    const progressText = document.getElementById('progressText');
    const resultCard = document.getElementById('resultCard');
    const resultTitle = document.getElementById('resultTitle');
    const resultMessage = document.getElementById('resultMessage');
    const connectionStatus = document.getElementById('connectionStatus');

    const startBtn = document.getElementById('startBtn');
    const resetBtn = document.getElementById('resetBtn');
    const saveBtn = document.getElementById('saveBtn');

    // State
    let ws = null;
    let stream = null;
    let streamingInterval = null;
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');

    // Connect to WebSocket
    function connect() {
      connectionStatus.textContent = 'Connecting...';
      connectionStatus.className = 'connection-status disconnected';

      ws = new WebSocket(SERVER_URL);

      ws.onopen = () => {
        console.log('✓ Connected to TRACK D server');
        connectionStatus.textContent = '✓ Connected';
        connectionStatus.className = 'connection-status connected';
        statusEl.textContent = 'CONNECTED';
        instructionEl.textContent = 'Click "Start Verification" to begin';
      };

      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);

        if (data.type === 'connection') {
          console.log('Server:', data.message);
        } else if (data.type === 'error') {
          alert('Error: ' + data.message);
        } else if (data.type === 'save_result') {
          alert('✓ Result saved: ' + data.filename);
        } else {
          // Detection data
          updateUI(data);
        }
      };

      ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        connectionStatus.textContent = '✗ Connection Error';
        connectionStatus.className = 'connection-status disconnected';
      };

      ws.onclose = () => {
        console.log('Disconnected');
        connectionStatus.textContent = '✗ Disconnected';
        connectionStatus.className = 'connection-status disconnected';
        stopStreaming();
      };
    }

    // Start camera
    async function startCamera() {
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          video: {
            width: 800,
            height: 600,
            facingMode: 'user'
          }
        });

        video.srcObject = stream;
        await video.play();
        console.log('✓ Camera started');
      } catch (err) {
        console.error('Camera error:', err);
        alert('Could not access camera: ' + err.message);
      }
    }

    // Start streaming frames
    function startStreaming() {
      if (streamingInterval) return;

      console.log('Starting frame streaming at', FRAME_RATE, 'FPS');

      streamingInterval = setInterval(() => {
        if (!ws || ws.readyState !== WebSocket.OPEN) {
          stopStreaming();
          return;
        }

        // Capture frame
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        ctx.drawImage(video, 0, 0);

        // Convert to base64
        const frameData = canvas.toDataURL('image/jpeg', 0.8);

        // Send to server
        ws.send(JSON.stringify({
          type: 'frame',
          frame: frameData
        }));
      }, 1000 / FRAME_RATE);
    }

    // Stop streaming
    function stopStreaming() {
      if (streamingInterval) {
        clearInterval(streamingInterval);
        streamingInterval = null;
        console.log('Stopped frame streaming');
      }
    }

    // Update UI
    function updateUI(data) {
      // Status
      statusEl.textContent = data.status;
      statusEl.className = 'status ' + data.status.toLowerCase();

      // Instruction
      instructionEl.textContent = data.ui_elements.instruction;

      // Progress
      const progress = Math.round(data.ui_elements.progress);
      progressEl.style.width = progress + '%';
      progressText.textContent = progress + '%';

      // Scores
      document.getElementById('score-motion').textContent = data.scores.motion.toFixed(1) + '%';
      document.getElementById('score-texture').textContent = data.scores.texture.toFixed(1) + '%';
      document.getElementById('score-edge').textContent = data.scores.edge_density.toFixed(1) + '%';
      document.getElementById('score-color').textContent = data.scores.color_variance.toFixed(1) + '%';
      document.getElementById('score-consistency').textContent = data.scores.consistency.toFixed(1) + '%';
      document.getElementById('score-overall').textContent = data.scores.overall.toFixed(1) + '%';

      // Result
      if (data.result) {
        resultCard.style.display = 'block';

        if (data.result === 'LIVE') {
          resultCard.className = 'result-card';
          resultTitle.textContent = '✓ LIVE FINGER DETECTED';
          resultMessage.textContent = `Confidence: ${data.confidence.toFixed(1)}%`;
        } else {
          resultCard.className = 'result-card spoof';
          resultTitle.textContent = '✗ SPOOF DETECTED';
          resultMessage.textContent = data.attack_type
            ? data.attack_type.replace('_', ' ').toUpperCase()
            : 'Attack detected';
        }

        console.log('='.repeat(50));
        console.log('RESULT:', data.result);
        console.log('Confidence:', data.confidence.toFixed(2) + '%');
        if (data.attack_type) {
          console.log('Attack Type:', data.attack_type);
        }
        console.log('='.repeat(50));
      } else {
        resultCard.style.display = 'none';
      }
    }

    // Button handlers
    startBtn.onclick = async () => {
      if (!stream) {
        await startCamera();
      }

      if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ command: 'START_ANALYSIS' }));
        startStreaming();
      } else {
        alert('Not connected to server!');
      }
    };

    resetBtn.onclick = () => {
      if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ command: 'RESET' }));
        resultCard.style.display = 'none';
      }
    };

    saveBtn.onclick = () => {
      if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ command: 'SAVE_RESULT' }));
      }
    };

    // Initialize
    window.onload = () => {
      connect();
    };

    window.onbeforeunload = () => {
      stopStreaming();
      if (stream) {
        stream.getTracks().forEach(track => track.stop());
      }
      if (ws) ws.close();
    };
  </script>
</body>
</html>
```

---

## ⚛️ React Example

```jsx
import React, { useEffect, useState, useRef } from 'react';

const LivenessDetector = () => {
  const SERVER_URL = 'ws://34.61.165.1:8765';
  const FRAME_RATE = 10;

  const [ws, setWs] = useState(null);
  const [data, setData] = useState(null);
  const [connected, setConnected] = useState(false);

  const videoRef = useRef(null);
  const canvasRef = useRef(null);
  const streamRef = useRef(null);
  const intervalRef = useRef(null);

  // Connect to WebSocket
  useEffect(() => {
    const websocket = new WebSocket(SERVER_URL);

    websocket.onopen = () => {
      console.log('Connected to TRACK D server');
      setConnected(true);
    };

    websocket.onmessage = (event) => {
      const message = JSON.parse(event.data);

      if (!message.type || message.type === 'data') {
        setData(message);
      } else if (message.type === 'error') {
        alert('Error: ' + message.message);
      } else if (message.type === 'save_result') {
        alert('Result saved: ' + message.filename);
      }
    };

    websocket.onclose = () => {
      console.log('Disconnected');
      setConnected(false);
    };

    setWs(websocket);

    return () => {
      websocket.close();
      stopStreaming();
      stopCamera();
    };
  }, []);

  const startCamera = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { width: 800, height: 600, facingMode: 'user' }
      });

      videoRef.current.srcObject = stream;
      await videoRef.current.play();
      streamRef.current = stream;
    } catch (err) {
      alert('Camera error: ' + err.message);
    }
  };

  const stopCamera = () => {
    if (streamRef.current) {
      streamRef.current.getTracks().forEach(track => track.stop());
      streamRef.current = null;
    }
  };

  const startStreaming = () => {
    if (intervalRef.current) return;

    intervalRef.current = setInterval(() => {
      if (!ws || ws.readyState !== WebSocket.OPEN) return;

      const video = videoRef.current;
      const canvas = canvasRef.current;

      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;

      const ctx = canvas.getContext('2d');
      ctx.drawImage(video, 0, 0);

      const frameData = canvas.toDataURL('image/jpeg', 0.8);

      ws.send(JSON.stringify({
        type: 'frame',
        frame: frameData
      }));
    }, 1000 / FRAME_RATE);
  };

  const stopStreaming = () => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };

  const handleStart = async () => {
    if (!streamRef.current) {
      await startCamera();
    }

    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ command: 'START_ANALYSIS' }));
      startStreaming();
    }
  };

  const handleReset = () => {
    ws?.send(JSON.stringify({ command: 'RESET' }));
  };

  const handleSave = () => {
    ws?.send(JSON.stringify({ command: 'SAVE_RESULT' }));
  };

  return (
    <div style={{ maxWidth: 900, margin: '0 auto', padding: 20 }}>
      <h1>TRACK D - Liveness Detection</h1>

      <div style={{ marginBottom: 20 }}>
        <div style={{
          padding: 10,
          borderRadius: 8,
          background: connected ? '#d1fae5' : '#fee2e2',
          color: connected ? '#065f46' : '#991b1b',
          textAlign: 'center',
          fontWeight: 600
        }}>
          {connected ? '✓ Connected' : '✗ Disconnected'}
        </div>
      </div>

      <canvas ref={canvasRef} style={{ display: 'none' }} />

      <video
        ref={videoRef}
        autoPlay
        playsInline
        style={{
          width: '100%',
          maxWidth: 640,
          borderRadius: 10,
          border: '3px solid #ddd',
          display: 'block',
          margin: '0 auto 20px'
        }}
      />

      <div style={{
        background: '#f5f5f5',
        padding: 20,
        borderRadius: 10,
        marginBottom: 20
      }}>
        <div style={{
          fontSize: 24,
          fontWeight: 'bold',
          textAlign: 'center',
          marginBottom: 10,
          color: data?.status === 'LIVE' ? '#10b981'
               : data?.status === 'SPOOF' ? '#ef4444'
               : data?.status === 'ANALYZING' ? '#3b82f6'
               : '#999'
        }}>
          {data?.status || 'DISCONNECTED'}
        </div>

        <div style={{ textAlign: 'center', marginBottom: 15 }}>
          {data?.ui_elements?.instruction || 'Click Start to begin'}
        </div>

        <div style={{
          width: '100%',
          height: 30,
          background: '#e5e7eb',
          borderRadius: 15,
          overflow: 'hidden'
        }}>
          <div style={{
            width: `${data?.ui_elements?.progress || 0}%`,
            height: '100%',
            background: 'linear-gradient(90deg, #3b82f6, #8b5cf6)',
            transition: 'width 0.3s',
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            color: 'white',
            fontWeight: 'bold'
          }}>
            {Math.round(data?.ui_elements?.progress || 0)}%
          </div>
        </div>
      </div>

      <div style={{
        display: 'grid',
        gridTemplateColumns: 'repeat(auto-fit, minmax(150px, 1fr))',
        gap: 15,
        marginBottom: 20
      }}>
        {['motion', 'texture', 'edge_density', 'color_variance', 'consistency', 'overall'].map(key => (
          <div key={key} style={{
            background: '#f9fafb',
            padding: 15,
            borderRadius: 8,
            textAlign: 'center'
          }}>
            <div style={{ fontSize: 12, color: '#6b7280', textTransform: 'uppercase' }}>
              {key.replace('_', ' ')}
            </div>
            <div style={{ fontSize: 24, fontWeight: 'bold', color: '#3b82f6' }}>
              {data?.scores?.[key]?.toFixed(1) || 0}%
            </div>
          </div>
        ))}
      </div>

      {data?.result && (
        <div style={{
          background: data.result === 'LIVE'
            ? 'linear-gradient(135deg, #10b981, #059669)'
            : 'linear-gradient(135deg, #ef4444, #dc2626)',
          color: 'white',
          padding: 30,
          borderRadius: 15,
          textAlign: 'center',
          marginBottom: 20
        }}>
          <h2 style={{ fontSize: 36, marginBottom: 10 }}>
            {data.result === 'LIVE' ? '✓ LIVE DETECTED' : '✗ SPOOF DETECTED'}
          </h2>
          <p style={{ fontSize: 18 }}>
            {data.result === 'LIVE'
              ? `Confidence: ${data.confidence?.toFixed(1)}%`
              : data.attack_type?.replace('_', ' ').toUpperCase()
            }
          </p>
        </div>
      )}

      <div style={{
        display: 'flex',
        gap: 10,
        justifyContent: 'center',
        flexWrap: 'wrap'
      }}>
        <button
          onClick={handleStart}
          disabled={!connected}
          style={{
            padding: '12px 30px',
            background: connected ? '#3b82f6' : '#9ca3af',
            color: 'white',
            border: 'none',
            borderRadius: 8,
            fontSize: 16,
            fontWeight: 600,
            cursor: connected ? 'pointer' : 'not-allowed'
          }}
        >
          Start Verification
        </button>
        <button
          onClick={handleReset}
          style={{
            padding: '12px 30px',
            background: '#6b7280',
            color: 'white',
            border: 'none',
            borderRadius: 8,
            fontSize: 16,
            fontWeight: 600,
            cursor: 'pointer'
          }}
        >
          Reset
        </button>
        <button
          onClick={handleSave}
          style={{
            padding: '12px 30px',
            background: '#6b7280',
            color: 'white',
            border: 'none',
            borderRadius: 8,
            fontSize: 16,
            fontWeight: 600,
            cursor: 'pointer'
          }}
        >
          Save Result
        </button>
      </div>
    </div>
  );
};

export default LivenessDetector;
```

---

## 🚨 Error Handling

### Connection Errors

```javascript
ws.onerror = (error) => {
  console.error('WebSocket error:', error);

  // Show user-friendly message
  showError('Connection failed. Please check your internet connection.');
};

ws.onclose = (event) => {
  console.log('Connection closed:', event.code, event.reason);

  // Attempt reconnection
  setTimeout(() => {
    console.log('Reconnecting...');
    connect();
  }, 5000);
};
```

### Camera Errors

```javascript
try {
  const stream = await navigator.mediaDevices.getUserMedia({ video: true });
} catch (err) {
  if (err.name === 'NotAllowedError') {
    showError('Camera permission denied. Please allow camera access.');
  } else if (err.name === 'NotFoundError') {
    showError('No camera found. Please connect a camera.');
  } else {
    showError('Camera error: ' + err.message);
  }
}
```

### Message Parsing Errors

```javascript
ws.onmessage = (event) => {
  try {
    const data = JSON.parse(event.data);
    // Process data
  } catch (err) {
    console.error('Failed to parse message:', err);
  }
};
```

---

## 🧪 Testing

### 1. Test Connection

```javascript
const ws = new WebSocket('ws://34.61.165.1:8765');

ws.onopen = () => {
  console.log('✓ Connection successful');
  ws.close();
};

ws.onerror = () => {
  console.error('✗ Connection failed');
};
```

### 2. Test Camera Access

```javascript
navigator.mediaDevices.getUserMedia({ video: true })
  .then(stream => {
    console.log('✓ Camera access granted');
    stream.getTracks().forEach(track => track.stop());
  })
  .catch(err => {
    console.error('✗ Camera access denied:', err);
  });
```

### 3. Browser Compatibility

```javascript
// Check WebSocket support
if (!window.WebSocket) {
  alert('Your browser does not support WebSocket');
}

// Check getUserMedia support
if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
  alert('Your browser does not support camera access');
}
```

### 4. Network Test

Open browser console and run:

```javascript
fetch('http://34.61.165.1:8765')
  .then(() => console.log('✓ Server reachable'))
  .catch(() => console.error('✗ Server unreachable'));
```

---

## ✅ Production Checklist

### Before Going Live:

- [ ] **SSL/TLS:** Use `wss://` instead of `ws://` (secure WebSocket)
- [ ] **Domain Name:** Replace IP with domain (e.g., `wss://api.yourdomain.com:8765`)
- [ ] **Error Handling:** Implement reconnection logic
- [ ] **Loading States:** Show spinners during camera initialization
- [ ] **User Feedback:** Clear error messages for camera/connection issues
- [ ] **Mobile Responsive:** Test on mobile devices
- [ ] **Browser Testing:** Test on Chrome, Firefox, Safari, Edge
- [ ] **Performance:** Monitor frame rate and adjust if needed
- [ ] **Privacy:** Add camera permission explanations
- [ ] **Accessibility:** Add ARIA labels and keyboard navigation
- [ ] **Analytics:** Track success/failure rates

### Performance Optimization:

**Frame Rate:**
- Desktop: 10-15 FPS (recommended)
- Mobile: 5-10 FPS (better battery life)

**Image Quality:**
- High quality: 0.9 (90%)
- Balanced: 0.8 (80%) ← Recommended
- Low bandwidth: 0.6 (60%)

**Resolution:**
- Desktop: 800x600 (recommended)
- Mobile: 640x480

```javascript
// Adaptive frame rate
const isMobile = /iPhone|iPad|Android/i.test(navigator.userAgent);
const FRAME_RATE = isMobile ? 8 : 12;
const IMAGE_QUALITY = isMobile ? 0.7 : 0.8;
```

---

## 📞 Support & Contact

**Issues or Questions?**
- GitHub Issues: https://github.com/abhimanyu86/TRACK_D/issues
- Server Status: Check `ws://34.61.165.1:8765` connection

**Server Logs:**
```bash
gcloud compute ssh trackd-server --zone=us-central1-a --command="sudo journalctl -u trackd -n 50"
```

---

## 📄 API Reference Summary

### Commands (Send)
- `START_ANALYSIS` - Begin session
- `RESET` - Clear analysis
- `SAVE_RESULT` - Save result (LIVE only)
- `STOP_ANALYSIS` - End session

### Frame Data (Send)
```json
{ "type": "frame", "frame": "data:image/jpeg;base64,..." }
```

### Detection Data (Receive)
```json
{
  "status": "WAITING|ANALYZING|LIVE|SPOOF",
  "scores": { "motion": 85.5, "texture": 72.3, ... },
  "result": "LIVE|SPOOF|null",
  "attack_type": "photo_attack|screen_attack|...|null",
  "confidence": 82.3,
  "ui_elements": { "instruction": "...", "progress": 60.0 }
}
```

---

## 🎯 Quick Reference

**Server:** `ws://34.61.165.1:8765`
**Frame Rate:** 10 FPS (100ms interval)
**Image Format:** JPEG, quality 0.8
**Resolution:** 800x600 or 640x480
**Auto-restart:** 3 seconds after result
**Threshold:** ≥70% = LIVE, <70% = SPOOF

---

**Happy Coding! 🚀**

---

*Last Updated: 2026-01-18*
*Server Version: cloud-1.0*
*Documentation Version: 1.0*
