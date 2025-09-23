<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>AI Security Dashboard — Firebase Frontend</title>

  <!-- Tailwind (CDN) -->
  <link href="https://cdnjs.cloudflare.com/ajax/libs/tailwindcss/2.2.19/tailwind.min.css" rel="stylesheet">

  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>

  <style>
    :root{
      --bg: #0f172a;
      --card: #0b1220;
      --muted: #94a3b8;
      --accent: #2563eb;
    }
    html,body { height:100%; background:var(--bg); color:#e6eef8; font-family: Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial; }
    .card { background:var(--card); border:1px solid rgba(255,255,255,0.04); border-radius:12px; padding:14px; }
    .muted { color:var(--muted); font-size:0.95rem; }
    #streamWrap { position:relative; background:#000; display:flex; align-items:center; justify-content:center; min-height:220px; }
    #videoImg { max-width:100%; max-height:calc(60vh); display:block; object-fit:contain; }
    #overlayCanvas { position:absolute; left:0; top:0; pointer-events:none; }
    .log-entry { border-left:4px solid; padding:8px; margin-bottom:8px; background: rgba(255,255,255,0.02); border-radius:6px; }
    .btn { padding:8px 12px; border-radius:8px; cursor:pointer; border:none; color:white; font-weight:600; }
    .btn.primary { background:#2563eb; }
    .btn.warn { background:#ef4444; }
    .btn.ghost { background:#10b981; }
    /* ensure map container has fixed height so Leaflet renders fine */
    #miniMap { height:220px; border-radius:10px; }
    pre { color:#e6eef8; white-space:pre-wrap; word-wrap:break-word; }
    /* responsive tweaks */
    @media (max-width: 1024px) {
      #videoImg, #overlayCanvas { max-height: 45vh; }
    }
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
              <button id="recordBtn" class="btn primary" data-state="off">Start Recording</button>
              <button id="analyzeBtn" class="btn" style="background:#06b6d4">Request AI Analysis</button>
            </div>
          </div>

          <div id="streamWrap" class="w-full rounded overflow-hidden relative">
            <!-- We'll use an <img> for the baseframe and a canvas overlay for detections -->
            <img id="videoImg" alt="live frame" src="" />
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
                <button id="leftBtn" class="btn primary">← Left</button>
                <button id="rightBtn" class="btn primary">Right →</button>
              </div>
              <div class="flex gap-2">
                <button id="flashOnBtn" class="btn">Flash On</button>
                <button id="flashOffBtn" class="btn">Flash Off</button>
              </div>
              <div class="flex gap-2">
                <button id="startRecordDevice" class="btn">Start Recording (Device)</button>
                <button id="stopRecordDevice" class="btn">Stop Recording (Device)</button>
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
          <div id="miniMap" class="rounded mt-2"></div>
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

  <!-- Firebase (v8 compatibility used to match original code) -->
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-auth.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>

  <!-- Leaflet -->
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

  <script>
  /*****************************************************************
   * CONFIG - replace with your Firebase project config
   * Do NOT store any secrets (Gemini/Telegram/etc.) in client code.
   *****************************************************************/
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

  /* DOM refs */
  const videoImg = document.getElementById('videoImg');
  const overlayCanvas = document.getElementById('overlayCanvas');
  const cameraStatusEl = document.getElementById('cameraStatus');
  const lastUpdateEl = document.getElementById('lastUpdate');
  const logContainer = document.getElementById('logContainer');
  const analysisResultEl = document.getElementById('analysisResult');
  const chatBox = document.getElementById('chatBox');
  const gpsLatEl = document.getElementById('gpsLat'), gpsLonEl = document.getElementById('gpsLon');

  const signInBtn = document.getElementById('signInBtn');
  const signUpBtn = document.getElementById('signUpBtn');
  const signOutBtn = document.getElementById('signOutBtn');
  const emailInput = document.getElementById('email');
  const passwordInput = document.getElementById('password');
  const authArea = document.getElementById('authArea');
  const userArea = document.getElementById('userArea');
  const userEmailEl = document.getElementById('userEmail');

  /* Listener registry to detach on sign-out */
  let listeners = [];

  function attachListener(path, event = 'value', cb){
    const ref = db.ref(path);
    ref.on(event, cb);
    listeners.push({ref, event});
    return ref;
  }
  function detachDatabaseListeners(){
    listeners.forEach(o => {
      try { o.ref.off(o.event); } catch(e){}
    });
    listeners = [];
  }

  /* ========== Auth UI ========== */
  signInBtn.onclick = async () => {
    const email = emailInput.value.trim();
    const pwd = passwordInput.value;
    if (!email || !pwd) return alert('Email/password required.');
    try { await auth.signInWithEmailAndPassword(email, pwd); }
    catch(err){ alert('Sign in: ' + err.message); }
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

  auth.onAuthStateChanged(user => {
    if (user) {
      authArea.style.display = 'none';
      userArea.style.display = 'flex';
      userEmailEl.textContent = user.email || user.uid;
      attachDatabaseListeners(); // start realtime listeners when signed in
      // Leaflet map may be in a hidden container previously -> invalidate size
      setTimeout(() => { safeInvalidateMap(); }, 300);
    } else {
      authArea.style.display = 'flex';
      userArea.style.display = 'none';
      userEmailEl.textContent = '---';
      detachDatabaseListeners();
      cameraStatusEl.textContent = 'signed-out';
    }
  });

  /* ========== Map (Leaflet) ========== */
  const miniMap = L.map('miniMap', { zoomControl:false, attributionControl:false }).setView([6.5244,3.3792], 6);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(miniMap);
  let gpsMarker = L.marker([6.5244,3.3792]).addTo(miniMap);
  let gpsPolyline = L.polyline([], { color:'#8b5cf6' }).addTo(miniMap);

  /* Debounced invalidateSize helper to avoid repeated layout thrashing */
  let invalidateTimeout = null;
  function safeInvalidateMap(delay = 200){
    if (invalidateTimeout) clearTimeout(invalidateTimeout);
    invalidateTimeout = setTimeout(() => {
      try { miniMap.invalidateSize(); } catch(e){ console.warn('invalidateSize failed', e); }
      invalidateTimeout = null;
    }, delay);
  }

  /* ========== Core realtime listeners ========== */
  function attachDatabaseListeners(){
    // latest frame (base64 JPEG string)
    attachListener('latest_frame_base64', 'value', (snap) => {
      const val = snap.val();
      if (!val) { cameraStatusEl.textContent = 'no frame'; videoImg.src = ''; clearOverlay(); return; }
      videoImg.src = 'data:image/jpeg;base64,' + val;
      cameraStatusEl.textContent = 'online';
      lastUpdateEl.textContent = new Date().toLocaleString();
      // overlay sizing will be handled when image finishes loading
    });

    // detection results (array/object)
    attachListener('detection_results', 'value', (snap) => {
      drawDetections(snap.val() || []);
    });

    // system log (array/object)
    attachListener('system_log', 'value', (snap) => {
      renderLogs(snap.val() || []);
    });

    // analysis results (latest)
    attachListener('analysis_results/latest', 'value', (snap) => {
      const val = snap.val();
      if (!val) return;
      analysisResultEl.textContent = JSON.stringify(val, null, 2);
      addLogToUI('gemini_response', 'New AI analysis available.');
    });

    // chat responses
    attachListener('chat_responses', 'value', (snap) => {
      const val = snap.val(); if (!val) return;
      chatBox.innerHTML = '';
      Object.values(val).forEach(item => appendChatBubble(item.text || item.response || JSON.stringify(item)));
    });

    // gps route
    attachListener('gps_route', 'value', (snap) => {
      renderGpsMini(snap.val() || []);
    });

    // vehicle_state (optional)
    attachListener('vehicle_state', 'value', (snap) => {
      const val = snap.val();
      if (val && val.status) addLogToUI('info', 'Vehicle: ' + val.status);
    });
  }

  /* ========== UI helpers ========== */
  function renderLogs(logs){
    logContainer.innerHTML = '';
    if (!logs) return;
    const list = Array.isArray(logs) ? logs.slice(0,50) : Object.values(logs).slice(0,50);
    list.reverse().forEach(l => addLogToUI(l.type || 'info', l.message || JSON.stringify(l)));
  }

  function addLogToUI(type, message){
    const wrap = document.createElement('div');
    wrap.className = 'log-entry';
    wrap.style.borderLeftColor = type.includes('error') ? '#ef4444' : (type.includes('gemini') ? '#8b5cf6' : '#34d399');
    wrap.innerHTML = `<div class="text-sm font-semibold">${(type||'INFO').toUpperCase()} <span class="muted text-xs">${new Date().toLocaleTimeString()}</span></div>
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

  /* ========== Detection overlay drawing ========== */
  function clearOverlay(){
    try {
      const ctx = overlayCanvas.getContext('2d');
      ctx && ctx.clearRect(0,0,overlayCanvas.width, overlayCanvas.height);
    } catch(e){}
  }

  function syncOverlaySize(){
    const img = videoImg;
    const canvas = overlayCanvas;
    if (!img || !canvas) return;
    const rect = img.getBoundingClientRect();
    canvas.width = Math.max(1, Math.floor(rect.width));
    canvas.height = Math.max(1, Math.floor(rect.height));
    canvas.style.left = img.offsetLeft + 'px';
    canvas.style.top = img.offsetTop + 'px';
    canvas.style.width = rect.width + 'px';
    canvas.style.height = rect.height + 'px';
  }

  // Use requestAnimationFrame to avoid layout thrash when image loads
  function scheduleSyncOverlay(){
    if (typeof window._overlayRaf !== 'undefined') cancelAnimationFrame(window._overlayRaf);
    window._overlayRaf = requestAnimationFrame(() => { syncOverlaySize(); drawDetections(lastDetections || []); });
  }

  let lastDetections = [];
  function drawDetections(detections){
    try {
      lastDetections = Array.isArray(detections) ? detections : (detections ? Object.values(detections) : []);
      syncOverlaySize();
      const canvas = overlayCanvas;
      const ctx = canvas.getContext('2d');
      ctx.clearRect(0,0,canvas.width,canvas.height);
      if (!lastDetections || !lastDetections.length) return;
      const img = videoImg;
      const naturalW = img.naturalWidth || canvas.width;
      const naturalH = img.naturalHeight || canvas.height;
      const scaleX = canvas.width / naturalW;
      const scaleY = canvas.height / naturalH;
      lastDetections.forEach(d => {
        if (!d.bbox || d.bbox.length < 4) return;
        const [x1,y1,x2,y2] = d.bbox.map(Number);
        const w = Math.max(1, (x2 - x1) * scaleX);
        const h = Math.max(1, (y2 - y1) * scaleY);
        const x = x1 * scaleX;
        const y = y1 * scaleY;
        ctx.lineWidth = 2;
        ctx.strokeStyle = '#ff4d4f';
        ctx.strokeRect(x,y,w,h);
        ctx.fillStyle = '#ff4d4f';
        ctx.font = '14px sans-serif';
        const label = (d.class || 'obj');
        ctx.fillText(label, x + 6, Math.max(14, y + 14));
      });
    } catch(e){ console.error('drawDetections', e); }
  }

  /* Keep overlay updated when image loads / window resizes */
  videoImg.addEventListener('load', () => {
    scheduleSyncOverlay();
  });
  window.addEventListener('resize', () => {
    scheduleSyncOverlay();
    safeInvalidateMap(250);
  });

  /* ========== Commands & actions (push to DB) ========== */
  async function sendCommand(cmd){
    const user = auth.currentUser ? (auth.currentUser.email || auth.currentUser.uid) : 'web';
    const entry = { cmd, user, ts: Date.now() };
    try {
      await db.ref('command_queue').push(entry);
      addLogToUI('command', `Queued: ${cmd}`);
    } catch(e){ addLogToUI('error','Failed to queue command: '+(e.message||e)); }
  }

  // wired control buttons
  document.getElementById('leftBtn').addEventListener('click', () => sendCommand('left'));
  document.getElementById('rightBtn').addEventListener('click', () => sendCommand('right'));
  document.getElementById('flashOnBtn').addEventListener('click', () => sendCommand('flash_on'));
  document.getElementById('flashOffBtn').addEventListener('click', () => sendCommand('flash_off'));
  document.getElementById('startRecordDevice').addEventListener('click', () => sendCommand('start_record'));
  document.getElementById('stopRecordDevice').addEventListener('click', () => sendCommand('stop_record'));

  // Analyze buttons
  document.getElementById('analyzeBtn').addEventListener('click', async () => {
    try {
      const newRef = db.ref('analysis_requests').push();
      await newRef.set({ requested_by: auth.currentUser ? auth.currentUser.uid : 'web', ts: Date.now(), frame_ts: Date.now() });
      addLogToUI('info', 'Analysis requested: ' + newRef.key);
    } catch(e){ addLogToUI('error','Analysis request failed: '+e.message); }
  });
  document.getElementById('manualAnalyzeBtn').addEventListener('click', () => document.getElementById('analyzeBtn').click());
  document.getElementById('fetchAnalysisBtn').addEventListener('click', async () => {
    const snap = await db.ref('analysis_results/latest').once('value');
    const val = snap.val();
    if (val) { analysisResultEl.textContent = JSON.stringify(val, null, 2); addLogToUI('info','Fetched latest analysis'); }
    else addLogToUI('info','No analysis result present.');
  });

  /* ========== Snapshot (download) ========== */
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

  /* ========== Client recording (frames drawn into a canvas) ========== */
  let clientRecorder = null, clientCanvas = null, clientStreamInterval = null, recordChunks = [];
  document.getElementById('startClientRecord').addEventListener('click', startClientRecord);
  document.getElementById('stopClientRecord').addEventListener('click', stopClientRecord);

  function startClientRecord(){
    if (!videoImg.src) return alert('No video frame available.');
    // create canvas sized to displayed image natural dimensions if available else bounding box
    const w = videoImg.naturalWidth || videoImg.width || 640;
    const h = videoImg.naturalHeight || videoImg.height || 360;
    clientCanvas = document.createElement('canvas');
    clientCanvas.width = w;
    clientCanvas.height = h;
    const ctx = clientCanvas.getContext('2d');

    clientRecorder = new MediaRecorder(clientCanvas.captureStream(10), { mimeType: 'video/webm;codecs=vp9' });
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

    clientStreamInterval = setInterval(() => {
      try {
        ctx.drawImage(videoImg, 0, 0, clientCanvas.width, clientCanvas.height);
      } catch(e){}
    }, 100); // ~10fps
  }

  function stopClientRecord(){
    if (clientRecorder && clientRecorder.state !== 'inactive') clientRecorder.stop();
    if (clientStreamInterval) { clearInterval(clientStreamInterval); clientStreamInterval = null; }
  }

  /* ========== Chat (push request) ========== */
  document.getElementById('sendChatBtn').addEventListener('click', async () => {
    const text = document.getElementById('chatInput').value.trim();
    if (!text) return;
    const idRef = db.ref('chat_requests').push();
    await idRef.set({ text, from: auth.currentUser ? auth.currentUser.uid : 'web', ts: Date.now() });
    addLogToUI('chat_request', 'Chat request queued.');
    document.getElementById('chatInput').value = '';
  });
  document.getElementById('clearChatBtn').addEventListener('click', () => { chatBox.innerHTML = ''; });

  /* ========== GPS mini map rendering ========== */
  function renderGpsMini(routeArr){
    if (!routeArr || !routeArr.length) return;
    // accept either array of points or object keyed by id
    const ptsRaw = Array.isArray(routeArr) ? routeArr : Object.values(routeArr);
    const pts = ptsRaw.map(p => [parseFloat(p.latitude), parseFloat(p.longitude)]);
    if (!pts.length) return;
    gpsPolyline.setLatLngs(pts);
    const last = pts[pts.length-1];
    gpsMarker.setLatLng(last);
    try { miniMap.setView(last, Math.max(10, miniMap.getZoom())); } catch(e){}
    gpsLatEl.textContent = last[0].toFixed(6);
    gpsLonEl.textContent = last[1].toFixed(6);
    safeInvalidateMap();
  }

  document.getElementById('openGpsBtn').addEventListener('click', () => window.open('/gps', '_blank'));
  document.getElementById('resetRouteBtn').addEventListener('click', async () => {
    if (!confirm('Reset stored GPS route?')) return;
    await db.ref('gps_route').set([]);
    addLogToUI('info','GPS route reset requested.');
  });

  /* ========== Vehicle control (with confirmation for dangerous actions) ========== */
  document.getElementById('shutdownBtn').addEventListener('click', async () => {
    if (!confirm('SHUTDOWN engine? This is dangerous.')) return;
    await db.ref('vehicle_commands').push({ command: 'shutdown_car', by: auth.currentUser ? auth.currentUser.uid : 'web', ts: Date.now() });
    addLogToUI('command', 'Vehicle shutdown queued.');
  });
  document.getElementById('enableBtn').addEventListener('click', async () => {
    await db.ref('vehicle_commands').push({ command: 'enable_car', by: auth.currentUser ? auth.currentUser.uid : 'web', ts: Date.now() });
    addLogToUI('command', 'Vehicle enable queued.');
  });

  /* ========== Utility: detach on page hide to reduce DB read ========== */
  window.addEventListener('beforeunload', () => detachDatabaseListeners());

  /* ========== OPTIONAL: small improvement - heartbeat to show connection health ========== */
  // You can have the backend toggle /camera_heartbeat to show the camera is alive.
  attachListener('camera_heartbeat', 'value', (snap) => {
    const v = snap.val();
    // if backend writes timestamp, display recency
    if (!v) return;
    const ts = Number(v) || Date.now();
    const ageSec = Math.round((Date.now() - ts)/1000);
    cameraStatusEl.textContent = ageSec < 10 ? 'online' : `online (last ${ageSec}s)`;
  });

  /* ========== End of script ========== */
  </script>
</body>
</html>
