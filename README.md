<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>ESP32 Mission Planner - Firebase UI</title>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
<style>
    body { font-family: Arial, sans-serif; margin: 0; display: flex; flex-direction: column; height: 100vh; }
    #map { flex-grow: 1; height: 60vh; } /* Map takes most of the vertical space */
    #controls { padding: 15px; background-color: #f8f8f8; border-top: 1px solid #eee; }
    textarea { width: calc(100% - 10px); height: 100px; margin-top: 10px; padding: 5px; border: 1px solid #ccc; font-family: monospace; resize: vertical; }
    button { margin: 5px 5px 5px 0; padding: 8px 15px; background-color: #4CAF50; color: white; border: none; border-radius: 4px; cursor: pointer; }
    button:hover { background-color: #45a049; }
    #robot-status-display { margin-top: 10px; padding: 8px; background-color: #e0f2f7; border-left: 5px solid #2196F3; font-size: 0.9em; }
    .status-text { margin-right: 10px; }
</style>
</head>
<body>
<div id="map"></div>
<div id="controls">
    <h3>Robot Status:</h3>
    <div id="robot-status-display">Loading status...</div>

    <button id="clearBtn">Clear Map Markers</button>
    <button id="uploadMissionBtn">Upload Mission to Robot</button>
    <button id="fetchMissionBtn">Fetch Mission from Robot</button>
    <button id="refreshConfigBtn">Refresh Config</button>

    <h4>Mission JSON (editable)</h4>
    <textarea id="missionArea" placeholder='{"waypoints":[{"lat":6.52, "lon":3.37}]}'></textarea>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<!-- Firebase SDKs - make sure to use a compatible version -->
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>

<script>
    // --- YOUR FIREBASE CONFIGURATION ---
    // IMPORTANT: Replace these placeholders with your actual Firebase project configuration.
    // Go to your Firebase project in the console -> Project settings -> General.
    // Under "Your apps", select the web app. Copy the 'firebaseConfig' object.
const firebaseConfig = {
  apiKey: "AIzaSyADd3G_8WJFmiJmB2ewOYhs9IpuJRtTQ7A",
  authDomain: "navigator-59e90.firebaseapp.com",
  databaseURL: "https://navigator-59e90-default-rtdb.firebaseio.com",
  projectId: "navigator-59e90",
  storageBucket: "navigator-59e90.firebasestorage.app",
  messagingSenderId: "677281499595",
  appId: "1:677281499595:web:f0bafebed198b2eece4e4e"
};

    // Initialize Firebase
    firebase.initializeApp(firebaseConfig);
    const allowedEmail = "israelefe093@gmail.com";
    const allowedPassword = "ENG2002551"; // Must match the ID your ESP32 uses to identify itself

    const loginForm = document.getElementByid("loginForm");
    const dashboard = document.getElementByid("dashboard");
    const loginBtn = document.getElementByid("loginBtn");

    // --- LEAFLET MAP SETUP ---
    let map = L.map('map').setView([6.5244, 3.3792], 15); // Default view
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        maxZoom: 19,
        attribution: '© OpenStreetMap contributors'
    }).addTo(map);

    let markerGroup = L.layerGroup().addTo(map);
    let robotMarker = null; // To display the robot's current position

    // --- HELPER FUNCTIONS ---
    function addWaypointMarker(latlng, draggable = true) {
        let m = L.marker(latlng, { draggable: draggable }).addTo(markerGroup);
        if (draggable) {
            m.on('dragend', updateMissionJsonFromMarkers);
            m.on('click', function() {
                if (confirm('Remove this waypoint marker?')) {
                    markerGroup.removeLayer(m);
                    updateMissionJsonFromMarkers();
                }
            });
        }
        return m;
    }

    function updateMissionJsonFromMarkers() {
        let wps = [];
        markerGroup.eachLayer(function(layer) {
            // Only include draggable markers (user-added waypoints) in the mission JSON
            if (layer.options.draggable) {
                let p = layer.getLatLng();
                wps.push({ lat: +p.lat.toFixed(7), lon: +p.lng.toFixed(7) });
            }
        });
        let mission = {
            mission_id: 'UI-' + Date.now(),
            waypoints: wps
        };
        document.getElementById('missionArea').value = JSON.stringify(mission, null, 2);
    }

    // --- MAP INTERACTIONS ---
    map.on('click', function(e) {
        // Add new draggable waypoint markers on map click
        addWaypointMarker(e.latlng, true);
        updateMissionJsonFromMarkers();
    });

    // --- UI BUTTON HANDLERS ---
    document.getElementById('clearBtn').onclick = function() {
        // Clear only the draggable waypoint markers
        markerGroup.eachLayer(function(layer) {
            if (layer.options.draggable) {
                markerGroup.removeLayer(layer);
            }
        });
        updateMissionJsonFromMarkers(); // Update JSON to reflect cleared waypoints
    };

    document.getElementById('uploadMissionBtn').onclick = function() {
        let txt = document.getElementById('missionArea').value;
        try {
            const mission = JSON.parse(txt);
            // Push mission to Firebase
            robotRef.child('mission').set(mission)
                .then(() => alert('Mission uploaded to Firebase successfully!'))
                .catch(error => alert('Mission upload failed: ' + error.message));
        } catch (e) {
            alert('Invalid JSON in mission area: ' + e.message);
        }
    };

    document.getElementById('fetchMissionBtn').onclick = function() {
        // This will trigger the .on('value') listener below and update the UI
        robotRef.child('mission').once('value', (snapshot) => {
            if (!snapshot.exists()) {
                 alert('No mission found in Firebase for this robot.');
            } else {
                alert('Mission data fetched from Firebase.');
            }
        });
    };

    document.getElementById('refreshConfigBtn').onclick = function() {
        // Optionally fetch and display config here
        robotRef.child('config').once('value', (snapshot) => {
            const config = snapshot.val();
            if (config) {
                alert(`Current Config: BaseSpeed=${config.baseSpeed || 'N/A'}, SafeDistance=${config.safeDistance || 'N/A'}`);
            } else {
                alert('No config found in Firebase.');
            }
        }).catch(error => console.error("Error fetching config:", error));
    };


    // --- FIREBASE LISTENERS (REAL-TIME UPDATES) ---

    // Listener for Robot Status (e.g., GPS, heading, sensors)
    robotRef.child('status').on('value', (snapshot) => {
        const status = snapshot.val();
        const statusDisplay = document.getElementById('robot-status-display');
        if (status) {
            statusDisplay.innerHTML = `
                <span class="status-text">Lat: ${status.lat ? status.lat.toFixed(6) : 'N/A'}</span>
                <span class="status-text">Lng: ${status.lng ? status.lng.toFixed(6) : 'N/A'}</span>
                <span class="status-text">Heading: ${status.heading ? status.heading.toFixed(1) + '°' : 'N/A'}</span>
                <span class="status-text">Battery: ${status.battery ? status.battery + '%' : 'N/A'}</span>
                <span class="status-text">Ultrasonic: ${status.ultrasonic ? status.ultrasonic.toFixed(1) + 'cm' : 'N/A'}</span>
                <span class="status-text">Updated: ${status.timestamp ? new Date(status.timestamp).toLocaleTimeString() : 'N/A'}</span>
            `;

            // Update robot marker on the map
            if (status.lat && status.lng) {
                const robotLatLng = [status.lat, status.lng];
                if (!robotMarker) {
                    robotMarker = L.marker(robotLatLng, {
                        icon: L.divIcon({
                            className: 'custom-robot-icon',
                            html: '<div style="font-size: 24px; color: blue;">🤖</div>', // Emoji icon
                            iconSize: [24, 24],
                            iconAnchor: [12, 12]
                        })
                    }).addTo(map);
                } else {
                    robotMarker.setLatLng(robotLatLng);
                }
                // Optional: Rotate marker based on heading
                // robotMarker.setRotationAngle(status.heading); // Requires leaflet-rotatedmarker plugin
                map.panTo(robotLatLng); // Keep robot in view
            }

        } else {
            statusDisplay.innerText = "Robot status not available.";
            if (robotMarker) {
                map.removeLayer(robotMarker);
                robotMarker = null;
            }
        }
    });

    // Listener for Mission Data
    robotRef.child('mission').on('value', (snapshot) => {
        const mission = snapshot.val();
        // Clear previous waypoint markers (only the user-added draggable ones)
        markerGroup.eachLayer(function(layer) {
            if (layer.options.draggable) { // Only remove the ones we put there
                 markerGroup.removeLayer(layer);
            }
        });

        if (mission && mission.waypoints) {
            mission.waypoints.forEach(wp => {
                // Add non-draggable markers for the mission received from Firebase
                addWaypointMarker([wp.lat, wp.lon], false);
            });
            if (markerGroup.getLayers().length > 0) {
                map.fitBounds(markerGroup.getBounds()); // Adjust map view to fit all mission waypoints
            }
            document.getElementById('missionArea').value = JSON.stringify(mission, null, 2);
        } else {
            document.getElementById('missionArea').value = JSON.stringify({ waypoints: [] }, null, 2);
        }
    });

    // Initial load/display
    updateMissionJsonFromMarkers(); // Initialize missionArea if map already has markers
</script>
</body>
</html>
