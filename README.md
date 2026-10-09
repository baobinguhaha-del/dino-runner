<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dino Escape - Endless Runner</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            font-family: Arial, sans-serif;
            background-color: #000;
        }
        #canvas-container {
            width: 100vw;
            height: 100vh;
        }
        #ui {
            position: absolute;
            top: 20px;
            left: 20px;
            color: white;
            font-size: 24px;
            font-weight: bold;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
            pointer-events: none;
        }
        #game-over {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: white;
            text-align: center;
            background: rgba(0, 0, 0, 0.85);
            padding: 30px;
            border-radius: 15px;
            display: none;
            box-shadow: 0 0 20px rgba(255, 0, 0, 0.5);
        }
        #game-over h1 {
            margin: 0 0 10px 0;
            color: #ff4444;
            font-size: 40px;
        }
        #restart-btn {
            margin-top: 20px;
            padding: 12px 24px;
            font-size: 18px;
            background: #ff4444;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: background 0.2s;
        }
        #restart-btn:hover {
            background: #cc0000;
        }
    </style>
    <!-- Nhúng thư viện Three.js để vẽ đồ họa 3D -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>
    <div id="ui">Điểm: <span id="score">0</span></div>
    <div id="game-over">
        <h1>BẠN ĐÃ BỊ ĂN THỊT!</h1>
        <p>Điểm của bạn: <span id="final-score">0</span></p>
        <button id="restart-btn" onclick="resetGame()">Chơi lại</button>
    </div>
    <div id="canvas-container"></div>

    <script>
        // --- THIẾT LẬP CƠ BẢN ---
        const container = document.getElementById('canvas-container');
        const scoreEl = document.getElementById('score');
        const finalScoreEl = document.getElementById('final-score');
        const gameOverEl = document.getElementById('game-over');

        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x1a0f2e); // Bầu trời đêm nhẹ
        scene.fog = new THREE.Fog(0x1a0f2e, 20, 90);

        const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        container.appendChild(renderer.domElement);

        // --- ÁNH SÁNG ---
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);

        const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
        dirLight.position.set(10, 20, 10);
        dirLight.castShadow = true;
        dirLight.shadow.mapSize.width = 1024;
        dirLight.shadow.mapSize.height = 1024;
        scene.add(dirLight);

        // --- BIẾN TRẠNG THÁI GAME ---
        const LANES = [-3, 0, 3]; // 3 làn đường
        let currentLane = 1; // Làn giữa (index 1)
        let isJumping = false;
        let jumpVelocity = 0;
        const gravity = -0.015;
        
        let score = 0;
        let speed = 0.4;
        let isGameOver = false;
        let clock = new THREE.Clock();

        // --- TẠO MẶT ĐƯỜNG RUNNER ---
        const roadWidth = 10;
        const roadLength = 200;
        const roadGeo = new THREE.PlaneGeometry(roadWidth, roadLength);
        const roadMat = new THREE.MeshStandardMaterial({ color: 0x333333 });
        const road = new THREE.Mesh(roadGeo, roadMat);
        road.rotation.x = -Math.PI / 2;
        road.position.z = -roadLength / 2 + 10;
        road.receiveShadow = true;
        scene.add(road);

        // Vạch kẻ đường
        for(let i = 0; i < 20; i++) {
            const laneGeo = new THREE.PlaneGeometry(0.2, 4);
            const laneMat = new THREE.MeshBasicMaterial({ color: 0xffffff });
            
            const line1 = new THREE.Mesh(laneGeo, laneMat);
            line1.rotation.x = -Math.PI / 2;
            line1.position.set(-1.5, 0.01, -i * 10);
            scene.add(line1);

            const line2 = line1.clone();
            line2.position.x = 1.5;
            scene.add(line2);
        }

        // --- TẠO NHÂN VẬT CHÍNH ---
        const playerGroup = new THREE.Group();
        
        // Thân
        const bodyGeo = new THREE.BoxGeometry(0.8, 1.2, 0.6);
        const bodyMat = new THREE.MeshStandardMaterial({ color: 0x00ff88 });
        const body = new THREE.Mesh(bodyGeo, bodyMat);
        body.position.y = 0.8;
        body.castShadow = true;
        playerGroup.add(body);

        // Đầu
        const headGeo = new THREE.BoxGeometry(0.6, 0.6, 0.6);
        const headMat = new THREE.MeshStandardMaterial({ color: 0xffccaa });
        const head = new THREE.Mesh(headGeo, headMat);
        head.position.y = 1.7;
        head.castShadow = true;
        playerGroup.add(head);

        playerGroup.position.set(LANES[currentLane], 0, 0);
        scene.add(playerGroup);

        // --- TẠO KHỦNG LONG ĐUỔI THEO ---
        const dinoGroup = new THREE.Group();
        
        // Thân khủng long
        const dinoBodyGeo = new THREE.BoxGeometry(2, 2.5, 3);
        const dinoMat = new THREE.MeshStandardMaterial({ color: 0xff3300 });
        const dinoBody = new THREE.Mesh(dinoBodyGeo, dinoMat);
        dinoBody.position.y = 1.5;
        dinoBody.castShadow = true;
        dinoGroup.add(dinoBody);

        // Đầu khủng long
        const dinoHeadGeo = new THREE.BoxGeometry(1.6, 1.4, 2);
        const dinoHead = new THREE.Mesh(dinoHeadGeo, dinoMat);
        dinoHead.position.set(0, 3, -1);
        dinoHead.castShadow = true;
        dinoGroup.add(dinoHead);

        // Mắt khủng long
        const eyeGeo = new THREE.BoxGeometry(0.2, 0.2, 0.2);
        const eyeMat = new THREE.MeshBasicMaterial({ color: 0xffff00 });
        const leftEye = new THREE.Mesh(eyeGeo, eyeMat);
        leftEye.position.set(-0.85, 3.2, -1.5);
        const rightEye = leftEye.clone();
        rightEye.position.x = 0.85;
        dinoGroup.add(leftEye);
        dinoGroup.add(rightEye);

        dinoGroup.position.set(0, 0, 6); // Đứng phía sau người chơi
        scene.add(dinoGroup);

        // --- CHƯỚNG NGẠI VẬT & TẢO CHUYỂN ĐỘNG ---
        const obstacles = [];
        const obstacleTypes = ['box', 'tall_box'];

        function spawnObstacle() {
            if (isGameOver) return;

            const lane = Math.floor(Math.random() * 3);
            const type = obstacleTypes[Math.floor(Math.random() * obstacleTypes.length)];
            
            let geo, mat, mesh;
            if (type === 'box') {
                geo = new THREE.BoxGeometry(1.5, 1, 1.5);
                mat = new THREE.MeshStandardMaterial({ color: 0xeb9b34 });
                mesh = new THREE.Mesh(geo, mat);
                mesh.position.y = 0.5;
            } else {
                geo = new THREE.BoxGeometry(1.5, 2.5, 1.5);
                mat = new THREE.MeshStandardMaterial({ color: 0x990000 });
                mesh = new THREE.Mesh(geo, mat);
                mesh.position.y = 1.25;
            }

            mesh.position.x = LANES[lane];
            mesh.position.z = -100; // Xuất hiện từ xa
            mesh.castShadow = true;
            mesh.receiveShadow = true;

            scene.add(mesh);
            obstacles.push(mesh);

            // Tần suất tạo vật cản ngẫu nhiên
            const nextSpawn = Math.max(600, 1500 - speed * 1000);
            setTimeout(spawnObstacle, nextSpawn);
        }
        spawnObstacle();

        // --- ĐIỀU KHIỂN ---
        document.addEventListener('keydown', (e) => {
            if (isGameOver) return;

            if ((e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') && currentLane > 0) {
                currentLane--;
            } else if ((e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') && currentLane < 2) {
                currentLane++;
            } else if ((e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W' || e.key === ' ') && !isJumping) {
                isJumping = true;
                jumpVelocity = 0.3;
            }
        });

        // --- VÒNG LẶP GAME (GAME LOOP) ---
        function animate() {
            if (isGameOver) return;
            requestAnimationFrame(animate);

            const delta = clock.getDelta();

            // 1. Di chuyển nhân vật mượt mà theo làn
            playerGroup.position.x += (LANES[currentLane] - playerGroup.position.x) * 0.2;

            // 2. Xử lý nhảy
            if (isJumping) {
                playerGroup.position.y += jumpVelocity;
                jumpVelocity += gravity;

                if (playerGroup.position.y <= 0) {
                    playerGroup.position.y = 0;
                    isJumping = false;
                }
            }

            // 3. Khủng long lắc lư nhẹ khi chạy theo
            dinoGroup.position.x += (playerGroup.position.x - dinoGroup.position.x) * 0.05;
            dinoGroup.position.y = Math.sin(clock.getElapsedTime() * 10) * 0.15;

            // 4. Xử lý di chuyển và va chạm chướng ngại vật
            for (let i = obstacles.length - 1; i >= 0; i--) {
                const obs = obstacles[i];
                obs.position.z += speed * 40 * delta;

                // Kiểm tra va chạm (AABB Bounding Box)
                const playerBox = new THREE.Box3().setFromObject(playerGroup);
                const obsBox = new THREE.Box3().setFromObject(obs);

                if (playerBox.intersectsBox(obsBox)) {
                    endGame();
                }

                // Xóa vật cản khi đi qua khỏi màn hình
                if (obs.position.z > 10) {
                    scene.remove(obs);
                    obstacles.splice(i, 1);
                }
            }

            // 5. Cập nhật camera đi theo nhân vật
            camera.position.set(playerGroup.position.x * 0.3, playerGroup.position.y + 4, playerGroup.position.z + 8);
            camera.lookAt(playerGroup.position.x * 0.3, playerGroup.position.y + 1, playerGroup.position.z - 10);

            // 6. Tăng điểm & độ khó
            score += 1;
            scoreEl.innerText = Math.floor(score / 5);
            speed += 0.00005;

            renderer.render(scene, camera);
        }

        function endGame() {
            isGameOver = true;
            finalScoreEl.innerText = Math.floor(score / 5);
            gameOverEl.style.display = 'block';

            // Khủng long chồm lên bắt nhân vật
            dinoGroup.position.z = playerGroup.position.z + 1;
            dinoGroup.position.x = playerGroup.position.x;
        }

        function resetGame() {
            // Xóa toàn bộ vật cản cũ
            for (let obs of obstacles) {
                scene.remove(obs);
            }
            obstacles.length = 0;

            // Reset thông số
            currentLane = 1;
            playerGroup.position.set(LANES[currentLane], 0, 0);
            dinoGroup.position.set(0, 0, 6);
            speed = 0.4;
            score = 0;
            isGameOver = false;
            isJumping = false;

            gameOverEl.style.display = 'none';
            clock.start();
            animate();
        }

        // Tự động căn chỉnh khi thay đổi kích thước màn hình
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        // Khởi chạy game
        animate();
    </script>
</body>
</html>
