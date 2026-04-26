[ironman.html](https://github.com/user-attachments/files/27099280/ironman.html)
# NikolaDimitro.github.io<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FaceID Interactive Particles</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; font-family: 'Segoe UI', sans-serif; }
        canvas { display: block; position: absolute; top: 0; left: 0; z-index: 1; }
        
        /* UI Layer */
        #ui {
            position: absolute;
            bottom: 20px;
            left: 20px;
            color: white;
            z-index: 10;
            pointer-events: none;
            background: rgba(0,0,0,0.6);
            padding: 15px;
            border-radius: 12px;
            border: 1px solid rgba(255,255,255,0.1);
            backdrop-filter: blur(5px);
            transition: opacity 0.5s;
        }

        /* Lock Screen Layer */
        #lock-screen {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.85);
            z-index: 100;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            backdrop-filter: blur(10px);
            transition: opacity 0.8s ease, visibility 0.8s;
        }
        
        #lock-screen.unlocked {
            opacity: 0;
            visibility: hidden;
            pointer-events: none;
        }

        /* Scanner Visuals */
        .scanner-box {
            width: 200px;
            height: 200px;
            border: 2px solid #333;
            position: relative;
            box-shadow: 0 0 50px rgba(0, 255, 204, 0.1);
        }
        .scanner-box::before, .scanner-box::after {
            content: ''; position: absolute; width: 20px; height: 20px;
            border: 2px solid #00ffcc; transition: all 0.3s;
        }
        .scanner-box::before { top: -2px; left: -2px; border-right: 0; border-bottom: 0; }
        .scanner-box::after { bottom: -2px; right: -2px; border-left: 0; border-top: 0; }
        
        .scan-line {
            width: 100%; height: 2px; background: #00ffcc;
            position: absolute; top: 0;
            box-shadow: 0 0 10px #00ffcc;
            animation: scan 2s infinite linear;
        }

        h1 { color: #fff; font-weight: 300; margin-top: 20px; letter-spacing: 2px; }
        .status-text { color: #00ffcc; font-size: 14px; text-transform: uppercase; animation: blink 1.5s infinite; }

        @keyframes scan { 0% {top: 0%; opacity:0;} 50% {opacity:1;} 100% {top: 100%; opacity:0;} }
        @keyframes blink { 0%, 100% {opacity: 1;} 50% {opacity: 0.5;} }

        /* Video Preview */
        #video-container {
            position: absolute; top: 0; left: 0; width: 120px; z-index: 5;
            opacity: 0.4; pointer-events: none; transform: scaleX(-1);
        }
    </style>
    
    <!-- Three.js -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    
    <!-- MediaPipe: Hands AND Face Detection -->
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_detection/face_detection.js" crossorigin="anonymous"></script>
</head>
<body>

    <!-- Lock Screen / Face ID -->
    <div id="lock-screen">
        <div class="scanner-box">
            <div class="scan-line"></div>
        </div>
        <h1>SYSTEM LOCKED</h1>
        <div class="status-text">Waiting for Facial Recognition...</div>
    </div>

    <!-- Main UI -->
    <div id="ui">
        <h2 style="color:#00ffcc; margin:0 0 10px;">FaceID Active</h2>
        <p>🖐 <b>1 Hand:</b> Pinch Zoom / Tilt Rotate</p>
        <p>👐 <b>2 Hands:</b> Fly Inside (Expand)</p>
        <p>✊ <b>Fist:</b> Morph Shape</p>
        <div style="margin-top:10px; border-top:1px solid #444; padding-top:10px;">
            <p id="shape-status" style="font-weight:bold; color:#fff">Shape: Sphere</p>
        </div>
    </div>

    <video id="input_video" style="display:none"></video>
    <canvas id="output_canvas" id="video-container"></canvas>

    <script>
        // ==========================================
        // 1. STATE MANAGEMENT
        // ==========================================
        let isSystemLocked = true; // Default locked
        let lastFaceDetectTime = 0;
        
        const lockScreen = document.getElementById('lock-screen');
        const statusText = document.querySelector('.status-text');

        function setLockState(locked) {
            isSystemLocked = locked;
            if (locked) {
                lockScreen.classList.remove('unlocked');
                statusText.innerText = "Scanning for Face...";
            } else {
                lockScreen.classList.add('unlocked');
            }
        }

        // ==========================================
        // 2. SCENE SETUP
        // ==========================================
        const scene = new THREE.Scene();
        scene.fog = new THREE.FogExp2(0x000000, 0.001);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 2000);
        camera.position.z = 40;

        const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        document.body.appendChild(renderer.domElement);

        // ==========================================
        // 3. PARTICLE SYSTEM
        // ==========================================
        const PARTICLE_COUNT = 6000;
        const geometry = new THREE.BufferGeometry();
        const positions = new Float32Array(PARTICLE_COUNT * 3);
        const colors = new Float32Array(PARTICLE_COUNT * 3);
        
        const currentPositions = [];
        const targetPositions = [];

        for (let i = 0; i < PARTICLE_COUNT; i++) {
            const x = (Math.random() - 0.5) * 60;
            const y = (Math.random() - 0.5) * 60;
            const z = (Math.random() - 0.5) * 60;
            
            positions[i * 3] = x;
            positions[i * 3 + 1] = y;
            positions[i * 3 + 2] = z;

            currentPositions.push({ x, y, z });
            targetPositions.push({ x, y, z });

            colors[i * 3] = 0; colors[i * 3 + 1] = 0; colors[i * 3 + 2] = 0; // Start dark
        }

        geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

        const sprite = new THREE.TextureLoader().load('https://threejs.org/examples/textures/sprites/snowflake1.png');
        
        const material = new THREE.PointsMaterial({
            size: 0.5,
            vertexColors: true,
            map: sprite,
            blending: THREE.AdditiveBlending,
            depthWrite: false,
            transparent: true,
            opacity: 0.9
        });

        const particles = new THREE.Points(geometry, material);
        scene.add(particles);

        // SHAPES
        const shapes = ['Sphere', 'Cube', 'Saturn', 'DNA', 'Torus'];
        let currentShapeIndex = 0;

        function getPoint(type, i) {
            const idx = i;
            const count = PARTICLE_COUNT;
            if (type === 'Sphere') {
                const r = 12;
                const phi = Math.acos(-1 + (2 * idx) / count);
                const theta = Math.sqrt(count * Math.PI) * phi;
                return { x: r * Math.cos(theta) * Math.sin(phi), y: r * Math.sin(theta) * Math.sin(phi), z: r * Math.cos(phi) };
            } else if (type === 'Cube') {
                const s = 15;
                return { x: (Math.random()-0.5)*s*2, y: (Math.random()-0.5)*s*2, z: (Math.random()-0.5)*s*2 };
            } else if (type === 'Saturn') {
                if (idx < count*0.6) {
                    const r = 7;
                    const phi = Math.acos(-1 + (2 * idx / (count*0.6)) / 1);
                    const theta = Math.sqrt((count*0.6) * Math.PI) * phi;
                    return { x: r * Math.cos(theta)*Math.sin(phi), y: r * Math.sin(theta)*Math.sin(phi), z: r * Math.cos(phi) };
                } else {
                    const angle = (idx / (count*0.4)) * Math.PI * 2 * 5;
                    const rad = 10 + Math.random() * 6;
                    return { x: rad * Math.cos(angle), y: (Math.random()-0.5), z: rad * Math.sin(angle) };
                }
            } else if (type === 'DNA') {
                const t = (idx / count) * 50;
                const radius = 6;
                const strand = idx % 2 === 0 ? 1 : -1; 
                return { x: (t - 25) * 1.5, y: Math.sin(t)*radius*strand, z: Math.cos(t)*radius*strand };
            } else if (type === 'Torus') {
                 const u = Math.random() * Math.PI * 2;
                 const v = Math.random() * Math.PI * 2;
                 const R = 10; const r = 3;
                 return { x: (R + r * Math.cos(v)) * Math.cos(u), y: (R + r * Math.cos(v)) * Math.sin(u), z: r * Math.sin(v) };
            }
            return {x:0,y:0,z:0};
        }

        function updateTargetShape(shapeName) {
            document.getElementById('shape-status').innerText = `Shape: ${shapeName}`;
            for (let i = 0; i < PARTICLE_COUNT; i++) {
                const p = getPoint(shapeName, i);
                targetPositions[i].x = p.x;
                targetPositions[i].y = p.y;
                targetPositions[i].z = p.z;
            }
        }

        // ==========================================
        // 4. AI & CAMERA LOGIC
        // ==========================================
        const videoElement = document.getElementById('input_video');
        
        // --- A. FACE DETECTION CONFIG ---
        const faceDetection = new FaceDetection({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_detection/${file}`});
        faceDetection.setOptions({ minDetectionConfidence: 0.5, model: 'short' });
        faceDetection.onResults(onFaceResults);

        // --- B. HAND TRACKING CONFIG ---
        const hands = new Hands({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`});
        hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.5, minTrackingConfidence: 0.5 });
        hands.onResults(onHandResults);

        // --- C. CAMERA FEED (Cascading Logic) ---
        const cameraFeed = new Camera(videoElement, {
            onFrame: async () => {
                // 1. Run Face Detection
                await faceDetection.send({image: videoElement});
                
                // 2. Only run Hands if unlocked to save resources
                if (!isSystemLocked) {
                    await hands.send({image: videoElement});
                }
            },
            width: 640,
            height: 480
        });
        cameraFeed.start();


        // --- D. RESULTS HANDLERS ---
        
        function onFaceResults(results) {
            // Check if any face detected
            if (results.detections.length > 0) {
                lastFaceDetectTime = Date.now();
                if (isSystemLocked) {
                    setLockState(false); // Unlock
                }
            } else {
                // Buffer to prevent locking on single dropped frame
                if (!isSystemLocked && Date.now() - lastFaceDetectTime > 500) {
                    setLockState(true); // Lock
                    resetHandData(); // Reset particles
                }
            }
        }

        let handPosition = { x: 0, y: 0, z: 0 };
        let handScale = 1;
        let handRotationZ = 0; 
        let lastSwitchTime = 0;

        function resetHandData() {
            // Smoothly reset global variables so particles drift back
            // We handle the drift in the animation loop by checking isSystemLocked
        }

        function isFist(landmarks) {
            const isFingerDown = (tip, pip) => landmarks[tip].y > landmarks[pip].y;
            return isFingerDown(8,6) && isFingerDown(12,10) && isFingerDown(16,14) && isFingerDown(20,18);
        }

        function onHandResults(results) {
            if (isSystemLocked) return; // Ignore if locked

            const handCount = results.multiHandLandmarks ? results.multiHandLandmarks.length : 0;
            let targetX = 0, targetY = 0, targetScale = 1, targetRot = 0;
            let triggerSwitch = false;

            if (handCount === 1) {
                const lm = results.multiHandLandmarks[0];
                targetX = (0.5 - lm[9].x) * 50; 
                targetY = (0.5 - lm[9].y) * 40;

                const dx = lm[8].x - lm[4].x;
                const dy = lm[8].y - lm[4].y;
                const dist = Math.sqrt(dx*dx + dy*dy);
                targetScale = 0.5 + (dist * 15); 

                const rotDx = lm[9].x - lm[0].x;
                const rotDy = lm[9].y - lm[0].y;
                targetRot = -Math.atan2(rotDx, rotDy) + Math.PI; 
                if (targetRot > Math.PI) targetRot -= Math.PI * 2;

                if (isFist(lm)) triggerSwitch = true;
            } 
            else if (handCount === 2) {
                const h1 = results.multiHandLandmarks[0];
                const h2 = results.multiHandLandmarks[1];

                const midX = (h1[9].x + h2[9].x) / 2;
                const midY = (h1[9].y + h2[9].y) / 2;
                targetX = (0.5 - midX) * 50;
                targetY = (0.5 - midY) * 40;

                const dx = h1[9].x - h2[9].x;
                const dy = h1[9].y - h2[9].y;
                const dist = Math.sqrt(dx*dx + dy*dy);
                targetScale = 0.5 + (dist * 45); 

                const h1x = h1[9].x; const h1y = h1[9].y;
                const h2x = h2[9].x; const h2y = h2[9].y;
                targetRot = Math.atan2(h2y - h1y, h2x - h1x);

                if (isFist(h1) && isFist(h2)) triggerSwitch = true;
            }

            // Smooth Interpolation
            if (!isNaN(targetX)) handPosition.x += (targetX - handPosition.x) * 0.1;
            if (!isNaN(targetY)) handPosition.y += (targetY - handPosition.y) * 0.1;
            if (!isNaN(targetScale)) handScale += (targetScale - handScale) * 0.1;
            if (!isNaN(targetRot)) handRotationZ += (targetRot - handRotationZ) * 0.1;

            if (triggerSwitch && Date.now() - lastSwitchTime > 1500) {
                currentShapeIndex = (currentShapeIndex + 1) % shapes.length;
                updateTargetShape(shapes[currentShapeIndex]);
                lastSwitchTime = Date.now();
            }
        }

        // ==========================================
        // 5. ANIMATION LOOP
        // ==========================================
        const clock = new THREE.Clock();
        updateTargetShape('Sphere');

        function animate() {
            requestAnimationFrame(animate);

            const time = clock.getElapsedTime();
            const positionsArr = particles.geometry.attributes.position.array;
            const colorsArr = particles.geometry.attributes.color.array;

            // Handle Locking Logic in Animation
            let activeScale = handScale;
            
            if (isSystemLocked) {
                // If locked, force scale to 1 and position to center smoothly
                activeScale = 1;
                handPosition.x = handPosition.x * 0.95;
                handPosition.y = handPosition.y * 0.95;
                handRotationZ = handRotationZ * 0.95;
                
                // Also reset internal handScale for next unlock
                handScale = handScale + (1 - handScale) * 0.1; 
            }

            // Dynamic Size
            let displaySize = 0.6 + (activeScale * 0.2);
            material.size = displaySize;

            for (let i = 0; i < PARTICLE_COUNT; i++) {
                let tx = targetPositions[i].x * activeScale;
                let ty = targetPositions[i].y * activeScale;
                let tz = targetPositions[i].z * activeScale;

                tx += handPosition.x;
                ty += handPosition.y;

                // Physics Lerp
                const lerpFactor = 0.08;
                currentPositions[i].x += (tx - currentPositions[i].x) * lerpFactor;
                currentPositions[i].y += (ty - currentPositions[i].y) * lerpFactor;
                currentPositions[i].z += (tz - currentPositions[i].z) * lerpFactor;

                // Noise
                const noiseAmt = 0.02 * activeScale;
                currentPositions[i].x += (Math.random() - 0.5) * noiseAmt;
                currentPositions[i].y += (Math.random() - 0.5) * noiseAmt;
                currentPositions[i].z += (Math.random() - 0.5) * noiseAmt;

                positionsArr[i * 3] = currentPositions[i].x;
                positionsArr[i * 3 + 1] = currentPositions[i].y;
                positionsArr[i * 3 + 2] = currentPositions[i].z;

                // Color & Brightness
                const dist = Math.sqrt(tx*tx + ty*ty);
                const hue = (time * 0.05 + (dist * 0.02) + (i * 0.0001)) % 1;
                
                let lightness = 0.5 + (activeScale * 0.04);
                if (lightness > 0.95) lightness = 0.95;
                
                // If locked, dim particles
                if (isSystemLocked) lightness *= 0.3;

                const color = new THREE.Color();
                color.setHSL(hue, 1.0, lightness);

                colorsArr[i * 3] = color.r;
                colorsArr[i * 3 + 1] = color.g;
                colorsArr[i * 3 + 2] = color.b;
            }

            particles.geometry.attributes.position.needsUpdate = true;
            particles.geometry.attributes.color.needsUpdate = true;
            
            particles.rotation.z = handRotationZ;
            particles.rotation.y = time * 0.05; 

            renderer.render(scene, camera);
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        animate();

    </script>
</body>
</html>
