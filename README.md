<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Jarvis 3D Gesture - Drawing Mode</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; color: #00d4ff; font-family: 'Courier New', Courier, monospace; }
        #canvas-container { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }
        #ui-layer { position: absolute; top: 20px; left: 20px; pointer-events: none; }
        .status { font-size: 1.2rem; text-shadow: 0 0 10px #00d4ff; }
        video { display: none; } /* Hide raw webcam */
    </style>
</head>
<body>

    <div id="ui-layer">
        <div class="status">JARVIS SYSTEM: ACTIVE</div>
        <div id="gesture-info">GESTURE: NONE</div>
        <div style="font-size: 0.8rem; opacity: 0.7;">Pinch to Draw | Open Palm to Move</div>
    </div>

    <div id="canvas-container"></div>

    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

    <script>
        // --- THREE.JS SETUP ---
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('canvas-container').appendChild(renderer.domElement);

        // Jarvis Grid & Lights
        const grid = new THREE.GridHelper(20, 20, 0x00d4ff, 0x002233);
        scene.add(grid);
        const light = new THREE.PointLight(0xffffff, 1, 100);
        light.position.set(0, 5, 5);
        scene.add(light);
        camera.position.z = 5;

        // --- DRAWING LOGIC VARIABLES ---
        let drawingPoints = [];
        let currentLine = null;
        let isPinching = false;

        // Function to Create Real 3D Material
        function startNewLine() {
            const material = new THREE.MeshStandardMaterial({ color: 0x00d4ff, emissive: 0x00d4ff });
            const geometry = new THREE.SphereGeometry(0.05, 8, 8); // Drawing with small spheres for "real" feel
            currentLine = new THREE.Group();
            scene.add(currentLine);
        }

        // --- MEDIAPIPE HANDS SETUP ---
        const hands = new Hands({
            locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
        });

        hands.setOptions({
            maxNumHands: 1,
            modelComplexity: 1,
            minDetectionConfidence: 0.7,
            minTrackingConfidence: 0.7
        });

        hands.onResults((results) => {
            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                const landmarks = results.multiHandLandmarks[0];
                
                // Index Finger Tip (Landmark 8) and Thumb Tip (Landmark 4)
                const indexTip = landmarks[8];
                const thumbTip = landmarks[4];

                // Map landmarks to 3D Space
                const x = (0.5 - indexTip.x) * 10;
                const y = (0.5 - indexTip.y) * 10;
                const z = -indexTip.z * 10;

                // Calculate distance for Pinch (Drawing Gesture)
                const distance = Math.sqrt(
                    Math.pow(indexTip.x - thumbTip.x, 2) + 
                    Math.pow(indexTip.y - thumbTip.y, 2)
                );

                document.getElementById('gesture-info').innerText = distance < 0.05 ? "GESTURE: DRAWING" : "GESTURE: TRACKING";

                if (distance < 0.05) { // Pinch detected
                    if (!isPinching) {
                        isPinching = true;
                        startNewLine();
                    }
                    // Add "Real" 3D object at finger position
                    const dotGeo = new THREE.SphereGeometry(0.06, 12, 12);
                    const dotMat = new THREE.MeshStandardMaterial({ color: 0x00d4ff });
                    const dot = new THREE.Mesh(dotGeo, dotMat);
                    dot.position.set(x, y, 0); // Drawing on a 2D plane in 3D space
                    currentLine.add(dot);
                } else {
                    isPinching = false;
                }
            }
        });

        const videoElement = document.createElement('video');
        const cameraUtils = new Camera({
            onFrame: async () => {
                await hands.send({ image: videoElement });
            },
            width: 640,
            height: 480
        });
        cameraUtils.start();

        // --- ANIMATION LOOP ---
        function animate() {
            requestAnimationFrame(animate);
            renderer.render(scene, camera);
        }
        animate();

        // Handle Window Resize
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
