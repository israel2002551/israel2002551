<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>AI Security Dashboard — Firebase Frontend</title>

<!-- Styles -->
<link href="https://cdnjs.cloudflare.com/ajax/libs/tailwindcss/2.2.19/tailwind.min.css" rel="stylesheet">
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
<style>
  body { background:#0f172a; color:#e6eef8; }
  .card { background:#0b1220; border:1px solid rgba(255,255,255,0.04); border-radius:12px; padding:12px; }
  .muted { color:#94a3b8; font-size:0.95rem; }
  #streamWrap { position:relative; background:#000; display:flex; align-items:center; justify-content:center; }
  #videoCanvas, #videoImg { max-width:100%; max-height:calc(60vh); display:block; }
  #overlayCanvas { position:absolute; left:0; top:0; pointer-events:none; }
  .log-entry { border-left:4px solid; padding:8px; margin-bottom:8px; background: rgba(255,255,255,0.02); border-radius:6px; }
  .btn { padding:8px 12px; border-radius:8px; cursor:pointer; border:none; color:white; }
  .btn.primary { background:#2563eb; }
  .btn.warn { background:#ef4444; }
  .btn.ghost { background:#10b981; }
</style>
</head>
<body class="p-6">

<div class="max-w-7xl mx-auto">

  <!-- Header -->
  <header class="flex items-center justify-between mb-6">
    <div class="flex items-center gap-4">
      <div class="p-3 bg-blue-600 rounded-lg">
        <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="white"><path d="M14.5 4h-5L7 7H4a2 2 0 0 0-2 2v9a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V9a2 2 0 0 0-2-2h-3l-2.5-3z"/><circle cx="12" cy="13" r="3"/></svg>
      </div>
      <div>
        <h1 class="text-xl font-bold">AI Security Dashboard (Firebase)</h1>
        <div class="muted">Client-only UI — requires a secure backend (Cloud Function) to perform Gemini/Telegram actions.</div>
      </div>
    </div>

    <div class="flex items-center gap-3">
      <div id="authArea" class="flex items-center gap-3">
        <input id="email" class="px-3 py-2 rounded bg-gray-800 border border-gray-700" placeholder="email" />
        <input id="password" type="password" class="px-3 py-2 rounded bg-gray-800 border border-gray-700" placeholder="password" />
        <button id="signInBtn" class="btn primary">Sign In</button>
        <button id="signUpBtn" class="btn ghost">Sign Up</button>
      </div>
      <div id="userArea" style="display:none" class="flex items-center gap-3">
        <div class="muted" id="userEmail">---</div>
        <button id="signOutBtn" class="btn warn">Sign out</button>
      </div>
    </div>
  </header>

  <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">

    <!-- Main stream + controls -->
    <section class="lg:col-span-2 space-y-6">
      <div class="card">
        <div class="flex justify-between items-start mb-3">
          <div>
            <h2 class="text-lg font-semibold">Live Security Feed</h2>
            <div class="muted">Frame & detection overlays (source: Firebase RTDB)</div>
          </div>
          <div class="flex gap-2">
            <button id="snapBtn" class="btn ghost">Capture Image</button>
            <button id="recordBtn" class="btn primary">Start Recording</button>
            <button id="analyzeBtn" class="btn" style="background:#06b6d4">Request AI Analysis</button>
          </div>
        </div>

        <div id="streamWrap" class="w-full rounded overflow-hidden relative">
          <!-- We'll use an <img> for the baseframe and a canvas overlay for detections -->
          <img id="videoImg" alt="live frame" src="" style="display:block; width:100%; height:auto; object-fit:contain;" />
          <canvas id="overlayCanvas"></canvas>
        </div>

        <div class="flex items-center justify-between mt-3">
          <div class="muted">Status: <span id="cameraStatus">loading</span></div>
          <div class="muted">Last update: <span id="lastUpdate">never</span></div>
        </div>
      </div>

      <!-- Controls & actions -->
      <div class="card grid grid-cols-1 md:grid-cols-3 gap-4">
        <div>
          <h3 class="font-semibold">Controls</h3>
          <div class="mt-2 flex flex-col gap-2">
            <div class="flex gap-2">
              <button onclick="sendCommand('left')" class="btn primary">← Left</button>
              <button onclick="sendCommand('right')" class="btn primary">Right →</button>
            </div>
            <div class="flex gap-2">
              <button onclick="sendCommand('flash_on')" class="btn">Flash On</button>
              <button onclick="sendCommand('flash_off')" class="btn">Flash Off</button>
            </div>
            <div class="flex gap-2">
              <button onclick="sendCommand('start_record')" class="btn">Start Recording (Device)</button>
              <button onclick="sendCommand('stop_record')" class="btn">Stop Recording (Device)</button>
            </div>
          </div>
        </div>

        <div>
          <h3 class="font-semibold">Analysis</h3>
          <div class="mt-2">
            <div class="muted">Manual AI analysis triggers an entry in the database. A backend Cloud Function should process it and write results to <code>/analysis_results/latest</code>.</div>
            <div class="mt-2 flex gap-2">
              <button id="manualAnalyzeBtn" class="btn" style="background:#8b5cf6">Trigger Manual Analysis</button>
              <button id="fetchAnalysisBtn" class="btn" style="background:#22c55e">Get Latest Result</button>
            </div>
            <pre id="analysisResult" class="mt-2 bg-gray-900 p-2 rounded text-sm" style="max-height:120px; overflow:auto;"></pre>
          </div>
        </div>

        <div>
          <h3 class="font-semibold">Client Recording</h3>
          <div class="muted">Record video in the browser (frames captured from image stream).</div>
          <div class="mt-2 flex flex-col gap-2">
            <button id="startClientRecord" class="btn primary">Start Client Record</button>
            <button id="stopClientRecord" class="btn warn">Stop & Download</button>
          </div>
        </div>
      </div>

      <!-- Chat area -->
      <div class="card">
        <h3 class="font-semibold">AI Chat Assistant</h3>
        <div class="mt-2 muted">Messages are posted to <code>/chat_requests</code>. A backend must process and write responses to <code>/chat_responses</code>.</div>
        <div class="mt-3 grid grid-cols-1 md:grid-cols-3 gap-3">
          <input id="chatInput" placeholder="Ask something about current frame..." class="px-3 py-2 rounded bg-gray-800 border"/>
          <button id="sendChatBtn" class="btn primary">Send</button>
          <button id="clearChatBtn" class="btn ghost">Clear Chat</button>
        </div>
        <div id="chatBox" class="mt-3 bg-gray-900 p-3 rounded" style="max-height:220px; overflow:auto"></div>
      </div>
    </section>

    <!-- Right column: logs, GPS quick, vehicle control -->
    <aside class="space-y-6">
      <div class="card">
        <h3 class="font-semibold">System Activity Log</h3>
        <div id="logContainer" class="mt-3" style="max-height:360px; overflow:auto"></div>
      </div>

      <div class="card">
        <h3 class="font-semibold">GPS Quick View</h3>
        <div id="miniMap" style="height:200px" class="rounded mt-2"></div>
        <div class="mt-2 muted">Latest latitude: <span id="gpsLat">N/A</span>, lon: <span id="gpsLon">N/A</span></div>
        <div class="mt-3 flex gap-2">
          <button id="openGpsBtn" class="btn ghost">Open GPS Page</button>
          <button id="resetRouteBtn" class="btn warn">Reset Route</button>
        </div>
      </div>

      <div class="card">
        <h3 class="font-semibold">Vehicle Control</h3>
        <div class="muted">Sends commands to <code>/vehicle_commands</code>. A secure backend/edge controller should read and act on these.</div>
        <div class="mt-3 flex gap-2">
          <button id="shutdownBtn" class="btn warn">SHUTDOWN ENGINE</button>
          <button id="enableBtn" class="btn primary">ENABLE ENGINE</button>
        </div>
      </div>
    </aside>
  </div>
</div>

<!-- Dependencies -->
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-auth.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<script>
/* ===========================================
   CONFIG - replace with your Firebase project config
   Get these values from Firebase Console -> Project Settings -> Your apps -> Config
   IMPORTANT: do NOT put your Gemini / Telegram secrets here.
   =========================================== */
const firebaseConfig = {
  apiKey: "REPLACE_WITH_YOUR_APIKEY",
  authDomain: "REPLACE_WITH_YOUR_AUTHDOMAIN",
  databaseURL: "REPLACE_WITH_YOUR_DATABASEURL",
  projectId: "REPLACE_WITH_YOUR_PROJECTID",
  storageBucket: "REPLACE_WITH_YOUR_STORAGEBUCKET",
  messagingSenderId: "REPLACE_WITH_MESSAGING_SENDER_ID",
  appId: "REPLACE_WITH_APPID"
};

firebase.initializeApp(firebaseConfig);
const auth = firebase.auth();
const db = firebase.database();

/* ===========================================
   DOM references
   =========================================== */
const videoImg = document.getElementById('videoImg');
const overlayCanvas = document.getElementById('overlayCanvas');
const cameraStatusEl = document.getElementById('cameraStatus');
const lastUpdateEl = document.getElementById('lastUpdate');
const logContainer = document.getElementById('logContainer');
const analysisResultEl = document.getElementById('analysisResult');
const chatBox = document.getElementById('chatBox');
const gpsLatEl = document.getElementById('gpsLat'), gpsLonEl = document.getElementById('gpsLon');

/* ===========================================
   Simple auth UI
   =========================================== */
const signInBtn = document.getElementById('signInBtn');
const signUpBtn = document.getElementById('signUpBtn');
const signOutBtn = document.getElementById('signOutBtn');
const emailInput = document.getElementById('email');
const passwordInput = document.getElementById('password');
const authArea = document.getElementById('authArea');
const userArea = document.getElementById('userArea');
const userEmailEl = document.getElementById('userEmail');

signInBtn.onclick = async () => {
  const email = emailInput.value.trim();
  const pwd = passwordInput.value;
  if (!email || !pwd) return alert('Email/password required.');
  try {
    await auth.signInWithEmailAndPassword(email, pwd);
  } catch(err){ alert('Sign in: ' + err.message); }
};

signUpBtn.onclick = async () => {
  const email = emailInput.value.trim();
  const pwd = passwordInput.value;
  if (!email || !pwd) return alert('Email/password required.');
  try {
    await auth.createUserWithEmailAndPassword(email, pwd);
    alert('Account created. You are signed in.');
  } catch(err){ alert('Sign up: ' + err.message); }
};

signOutBtn.onclick = async () => { await auth.signOut(); };

/* Show/hide UI on auth changes */
auth.onAuthStateChanged(user => {
  if (user) {
    authArea.style.display = 'none';
    userArea.style.display = 'flex';
    userEmailEl.textContent = user.email || user.uid;
    attachDatabaseListeners(); // start realtime listeners when signed in
  } else {
    authArea.style.display = 'flex';
    userArea.style.display = 'none';
    detachDatabaseListeners();
  }
});

/* ===========================================
   Database paths used by this frontend
   - latest_frame_base64: string (base64 JPEG bytes)
   - detection_results: array/object of detected items [{class, bbox:[x1,y1,x2,y2]}]
   - system_log: array of {timestamp, type, message}
   - command_queue: push children contain {cmd, user, ts}
   - analysis_requests: push to request AI analysis
   - analysis_results/latest: {analysis: "...", ts: ...}
   - chat_requests / chat_responses: for chat processing backend
   - gps_route: array of {latitude, longitude, timestamp}
   - vehicle_commands: push vehicle commands
   =========================================== */

let listeners = [];
function attachListener(refPath, cb) {
  const ref = db.ref(refPath);
  ref.on('value', snapshot => cb(snapshot.val()));
  listeners.push({ref, path: refPath});
}
function detachDatabaseListeners(){
  listeners.forEach(o => { try { o.ref.off(); } catch(e){} });
  listeners = [];
}

/* ===========================================
   Core realtime listeners (attach on auth)
   =========================================== */
function attachDatabaseListeners() {
  // latest frame (base64 JPEG string)
  attachListener('latest_frame_base64', (val) => {
    if (!val) { cameraStatusEl.textContent = 'no frame'; return; }
    videoImg.src = 'data:image/jpeg;base64,' + val;
    cameraStatusEl.textContent = 'online';
    lastUpdateEl.textContent = new Date().toLocaleTimeString();

    // resize overlay
    setTimeout(syncOverlaySize, 50);
  });

  // detection results (array or object)
  attachListener('detection_results', (val) => {
    drawDetections(val || []);
  });

  // system log (array)
  attachListener('system_log', (val) => {
    renderLogs(val || []);
  });

  // analysis results (latest)
  attachListener('analysis_results/latest', (val) => {
    if (!val) return;
    analysisResultEl.textContent = JSON.stringify(val, null, 2);
    addLogToUI('gemini_response', 'New AI analysis available.');
  });

  // chat responses
  attachListener('chat_responses', (val) => {
    // expect object keyed by id; append any responses
    if (!val) return;
    chatBox.innerHTML = '';
    Object.values(val).forEach(item => {
      appendChatBubble(item.text || item.response || JSON.stringify(item));
    });
  });

  // gps route
  attachListener('gps_route', (val) => {
    renderGpsMini(val || []);
  });

  // vehicle_state
  attachListener('vehicle_state', (val) => {
    // optional: render vehicle state
    if (val && val.status) addLogToUI('info', 'Vehicle: ' + val.status);
  });
}

/* ===========================================
   UI helpers: logs, detection overlays, chat
   =========================================== */
function renderLogs(logs){
  logContainer.innerHTML = '';
  if (!logs) return;
  // logs may be array-like; keep last 50
  const list = Array.isArray(logs) ? logs.slice(0,50) : Object.values(logs).slice(0,50);
  list.reverse().forEach(l => addLogToUI(l.type || 'info', l.message || JSON.stringify(l)));
}

function addLogToUI(type, message){
  const wrap = document.createElement('div');
  wrap.className = 'log-entry';
  wrap.style.borderLeftColor = type.includes('error') ? '#ef4444' : (type.includes('gemini') ? '#8b5cf6' : '#34d399');
  wrap.innerHTML = `<div class="text-sm font-semibold">${type.toUpperCase()} <span class="muted text-xs">${new Date().toLocaleTimeString()}</span></div>
                    <div class="muted mt-1">${message}</div>`;
  logContainer.prepend(wrap);
  while (logContainer.childElementCount > 200) logContainer.removeChild(logContainer.lastChild);
}

function appendChatBubble(text){
  const d = document.createElement('div');
  d.className = 'p-2 rounded mb-2';
  d.style.background = 'rgba(255,255,255,0.02)';
  d.textContent = text;
  chatBox.appendChild(d);
  chatBox.scrollTop = chatBox.scrollHeight;
}

/* ===========================================
   Draw detection boxes on top of image
   detection format expected: [{class: "person", bbox:[x1,y1,x2,y2]}, ...]
   Coordinates should be relative to original image pixel coordinates.
   =========================================== */
function syncOverlaySize(){
  const img = videoImg;
  const canvas = overlayCanvas;
  if (!img || !canvas) return;
  const rect = img.getBoundingClientRect();
  canvas.width = rect.width;
  canvas.height = rect.height;
  canvas.style.left = img.offsetLeft + 'px';
  canvas.style.top = img.offsetTop + 'px';
  canvas.style.width = rect.width + 'px';
  canvas.style.height = rect.height + 'px';
}
function drawDetections(detections){
  try {
    syncOverlaySize();
    const canvas = overlayCanvas;
    const ctx = canvas.getContext('2d');
    ctx.clearRect(0,0,canvas.width,canvas.height);
    if (!detections || !detections.length) return;
    // We assume detection.bbox coordinates refer to the same resolution as the original image.
    // If your backend provides image_width/image_height, use them to scale.
    const img = videoImg;
    const naturalW = img.naturalWidth || canvas.width;
    const naturalH = img.naturalHeight || canvas.height;
    const scaleX = canvas.width / naturalW;
    const scaleY = canvas.height / naturalH;
    detections.forEach(d => {
      if (!d.bbox || d.bbox.length < 4) return;
      const [x1,y1,x2,y2] = d.bbox;
      const w = (x2 - x1) * scaleX;
      const h = (y2 - y1) * scaleY;
      const x = x1 * scaleX;
      const y = y1 * scaleY;
      ctx.lineWidth = 2;
      ctx.strokeStyle = '#ff4d4f';
      ctx.strokeRect(x,y,w,h);
      ctx.fillStyle = '#ff4d4f';
      ctx.font = '14px sans-serif';
      const label = (d.class || 'obj');
      ctx.fillText(label, x + 4, Math.max(14, y + 14));
    });
  } catch(e){ console.error(e); }
}

/* ===========================================
   Commands & actions: write to database queue
   =========================================== */
async function sendCommand(cmd){
  const user = auth.currentUser ? (auth.currentUser.email || auth.currentUser.uid) : 'web';
  const entry = { cmd, user, ts: Date.now() };
  try {
    await db.ref('command_queue').push(entry);
    addLogToUI('command', `Queued: ${cmd}`);
  } catch(e){ addLogToUI('error','Failed to queue command: '+e.message); }
}

/* Trigger AI analysis by pushing request to /analysis_requests */
document.getElementById('analyzeBtn').addEventListener('click', async () => {
  try {
    const id = await db.ref('analysis_requests').push({ requested_by: auth.currentUser ? auth.currentUser.uid : 'web', ts: Date.now(), frame_ts: Date.now() }).key;
    addLogToUI('info', 'Analysis requested: '+id);
    // backend should pick up this request, analyze, and write result to /analysis_results/latest
  } catch(e){ addLogToUI('error','Analysis request failed: '+e.message); }
});

/* manual analyze button (same function) */
document.getElementById('manualAnalyzeBtn').addEventListener('click', () => document.getElementById('analyzeBtn').click());

/* Fetch latest analysis result (also auto-updates via listener) */
document.getElementById('fetchAnalysisBtn').addEventListener('click', async () => {
  const snap = await db.ref('analysis_results/latest').once('value');
  const val = snap.val();
  if (val) { analysisResultEl.textContent = JSON.stringify(val, null, 2); addLogToUI('info','Fetched latest analysis'); }
  else addLogToUI('info','No analysis result present.');
});

/* ===========================================
   Snap image: download current base64 image
   =========================================== */
document.getElementById('snapBtn').addEventListener('click', async () => {
  try {
    const snap = await db.ref('latest_frame_base64').once('value');
    const b = snap.val();
    if (!b) return alert('No frame available to capture.');
    const a = document.createElement('a');
    a.href = 'data:image/jpeg;base64,' + b;
    a.download = 'snapshot_' + new Date().toISOString() + '.jpg';
    document.body.appendChild(a); a.click(); a.remove();
    addLogToUI('info','Snapshot downloaded (client).');
  } catch(e){ addLogToUI('error','Snapshot failed: '+e.message); }
});

/* ===========================================
   Client-side recording: capture frames into MediaRecorder
   =========================================== */
let clientRecorder = null, clientCanvas = null, clientStream = null, recordChunks = [];
document.getElementById('startClientRecord').addEventListener('click', startClientRecord);
document.getElementById('stopClientRecord').addEventListener('click', stopClientRecord);

function startClientRecord(){
  if (!videoImg.src) return alert('No video frame available.');
  // create canvas the size of displayed image
  clientCanvas = document.createElement('canvas');
  clientCanvas.width = videoImg.naturalWidth || videoImg.width;
  clientCanvas.height = videoImg.naturalHeight || videoImg.height;
  const ctx = clientCanvas.getContext('2d');
  // draw frames at ~10fps
  clientRecorder = new MediaRecorder(clientCanvas.captureStream(10), { mimeType: 'video/webm' });
  recordChunks = [];
  clientRecorder.ondataavailable = (e) => { if (e.data.size) recordChunks.push(e.data); };
  clientRecorder.onstop = () => {
    const blob = new Blob(recordChunks, { type: 'video/webm' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a'); a.href = url; a.download = 'recording_'+Date.now()+'.webm'; document.body.appendChild(a); a.click(); a.remove();
    URL.revokeObjectURL(url); addLogToUI('info','Client recording downloaded.');
  };
  clientRecorder.start();
  addLogToUI('info','Client recording started.');
  // drawing interval
  clientStream = setInterval(() => {
    try {
      ctx.drawImage(videoImg, 0, 0, clientCanvas.width, clientCanvas.height);
    } catch(e){}
  }, 100);
}

function stopClientRecord(){
  if (clientRecorder && clientRecorder.state !== 'inactive') clientRecorder.stop();
  if (clientStream) { clearInterval(clientStream); clientStream = null; }
}

/* ===========================================
   Chat: push request and expect backend to respond
   =========================================== */
document.getElementById('sendChatBtn').addEventListener('click', async () => {
  const text = document.getElementById('chatInput').value.trim();
  if (!text) return;
  const idRef = db.ref('chat_requests').push();
  await idRef.set({ text, from: auth.currentUser ? auth.currentUser.uid : 'web', ts: Date.now() });
  addLogToUI('chat_request', 'Chat request queued.');
  document.getElementById('chatInput').value = '';
});

/* Clear chat box (UI only) */
document.getElementById('clearChatBtn').addEventListener('click', () => { chatBox.innerHTML = ''; });

/* ===========================================
   GPS mini map (Leaflet)
   =========================================== */
const miniMap = L.map('miniMap', { zoomControl:false, attributionControl:false }).setView([6.5244,3.3792], 6);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(miniMap);
let gpsMarker = L.marker([6.5244,3.3792]).addTo(miniMap);
let gpsPolyline = L.polyline([], { color:'#8b5cf6' }).addTo(miniMap);
function renderGpsMini(routeArr){
  if (!routeArr || !routeArr.length) return;
  const pts = routeArr.map(p => [parseFloat(p.latitude), parseFloat(p.longitude)]);
  gpsPolyline.setLatLngs(pts);
  const last = pts[pts.length-1];
  gpsMarker.setLatLng(last);
  miniMap.setView(last, Math.max(10, miniMap.getZoom()));
  gpsLatEl.textContent = last[0].toFixed(6);
  gpsLonEl.textContent = last[1].toFixed(6);
}

/* Buttons for opening GPS page and resetting route */
document.getElementById('openGpsBtn').addEventListener('click', () => window.open('/gps', '_blank'));
document.getElementById('resetRouteBtn').addEventListener('click', async () => {
  if (!confirm('Reset stored GPS route?')) return;
  await db.ref('gps_route').set([]);
  addLogToUI('info','GPS route reset requested.');
});

/* ===========================================
   Vehicle control
   =========================================== */
document.getElementById('shutdownBtn').addEventListener('click', async () => {
  if (!confirm('SHUTDOWN engine? This is dangerous.')) return;
  await db.ref('vehicle_commands').push({ command: 'shutdown_car', by: auth.currentUser ? auth.currentUser.uid : 'web', ts: Date.now() });
  addLogToUI('command', 'Vehicle shutdown queued.');
});
document.getElementById('enableBtn').addEventListener('click', async () => {
  await db.ref('vehicle_commands').push({ command: 'enable_car', by: auth.currentUser ? auth.currentUser.uid : 'web', ts: Date.now() });
  addLogToUI('command', 'Vehicle enable queued.');
});

/* ===========================================
   Detach listeners when signed out (handled above)
   =========================================== */

/* ===========================================
   Utility: detach on page hide to reduce DB read
   =========================================== */
window.addEventListener('beforeunload', () => detachDatabaseListeners());

/* Keep overlay updated when image loads / window resizes */
videoImg.addEventListener('load', syncOverlayDims);
window.addEventListener('resize', syncOverlayDims);

function syncOverlayDims(){
  const imgRect = videoImg.getBoundingClientRect();
  const canvas = overlayCanvas;
  canvas.width = imgRect.width;
  canvas.height = imgRect.height;
  canvas.style.left = videoImg.offsetLeft + 'px';
  canvas.style.top = videoImg.offsetTop + 'px';
  canvas.style.width = imgRect.width + 'px';
  canvas.style.height = imgRect.height + 'px';
}

</script>
</body>
</html>
