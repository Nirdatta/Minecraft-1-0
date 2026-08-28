
  <!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Minecraft Clone JS - Ottimizzato</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #87CEEB; font-family: 'Courier New', Courier, monospace; user-select: none; }
        #canvas-container { width: 100vw; height: 100vh; display: block; }
        
        #crosshair {
            position: absolute; top: 50%; left: 50%; width: 20px; height: 20px;
            transform: translate(-50%, -50%); pointer-events: none;
            color: white; font-size: 24px; text-align: center; line-height: 20px;
            text-shadow: 1px 1px 0 #000; z-index: 10;
        }
        #loading {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background-color: #222; color: #fff; display: flex;
            justify-content: center; align-items: center; font-size: 30px;
            z-index: 100; transition: opacity 0.5s;
        }
        #inventory {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            background: rgba(0,0,0,0.8); padding: 20px; border-radius: 10px;
            display: none; grid-template-columns: repeat(3, 1fr); gap: 10px; z-index: 20;
        }
        .block-btn {
            width: 60px; height: 60px; border: 2px solid #555; cursor: pointer;
            color: white; text-align: center; line-height: 60px; font-size: 12px;
        }
        .block-btn:hover, .block-btn.active { border-color: white; }
        #instructions {
            position: absolute; top: 10px; left: 10px; color: white;
            text-shadow: 1px 1px 0 #000; pointer-events: none; font-size: 14px;
        }
    </style>
</head>
<body>

    <div id="loading">Loading Terrain...</div>
    <div id="crosshair">+</div>
    <div id="instructions">
        Click: Gioca/Piazza/Rompi | WASD: Muovi | Spazio: Salta | Spazio x2: Volo | E: Inventario
    </div>
    
    <div id="inventory">
        <div class="block-btn" style="background:#5C4033" onclick="selectBlock(1, this)">Terra</div>
        <div class="block-btn" style="background:#808080" onclick="selectBlock(2, this)">Pietra</div>
        <div class="block-btn" style="background:#333333" onclick="selectBlock(3, this)">Bedrock</div>
        <div class="block-btn" style="background:#8B5A2B" onclick="selectBlock(4, this)">Legno</div>
        <div class="block-btn active" style="background:#ADD8E6; opacity:0.7" onclick="selectBlock(5, this)">Vetro</div>
        <div class="block-btn" style="background:#B22222" onclick="selectBlock(6, this)">Mattoni</div>
    </div>

    <div id="canvas-container"></div>

    <script src="https://unpkg.com/three@0.128.0/build/three.min.js"></script>
    <script src="https://unpkg.com/three@0.128.0/examples/js/controls/PointerLockControls.js"></script>

    <script>
        // 1. INIZIALIZZAZIONE SCENA
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87CEEB); 
        scene.fog = new THREE.Fog(0x87CEEB, 20, 50);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
        const renderer = new THREE.WebGLRenderer({ antialias: false });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('canvas-container').appendChild(renderer.domElement);

        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);
        const dirLight = new THREE.DirectionalLight(0xffffff, 0.6);
        dirLight.position.set(100, 200, 50);
        scene.add(dirLight);

        // Uso di THREE globale (risolve il crash dei moduli non caricati)
        const controls = new THREE.PointerLockControls(camera, document.body);
        scene.add(controls.getObject());

        // 2. MATERIALI
        const geometry = new THREE.BoxGeometry(1, 1, 1);
        const materials = {
            1: new THREE.MeshLambertMaterial({ color: 0x5C4033 }), 
            2: new THREE.MeshLambertMaterial({ color: 0x808080 }), 
            3: new THREE.MeshLambertMaterial({ color: 0x333333 }), 
            4: new THREE.MeshLambertMaterial({ color: 0x8B5A2B }), 
            5: new THREE.MeshLambertMaterial({ color: 0xADD8E6, transparent: true, opacity: 0.6 }), 
            6: new THREE.MeshLambertMaterial({ color: 0xB22222 }), 
            7: new THREE.MeshLambertMaterial({ color: 0x228B22 })  
        };

        let currentBlockType = 5;
        function selectBlock(type, el) {
            currentBlockType = type;
            document.querySelectorAll('.block-btn').forEach(b => b.classList.remove('active'));
            el.classList.add('active');
        }

        // 3. GENERAZIONE MONDO OTTIMIZZATA
        const chunkSize = 16;
        const renderDistance = 2; 
        const chunks = new Map(); 
        const blocks = new Map(); 
        const chunkQueue = []; // Coda per caricare il terreno un po' alla volta

        function getBlockKey(x, y, z) { return `${Math.round(x)},${Math.round(y)},${Math.round(z)}`; }
        function getChunkKey(cx, cz) { return `${cx},${cz}`; }

        function noise(x, z) {
            return (Math.sin(x * 0.1) + Math.cos(z * 0.1) + Math.sin(x * 0.05 + z * 0.05)) * 2;
        }

        function generateChunk(cx, cz) {
            const chunkKey = getChunkKey(cx, cz);
            if (chunks.has(chunkKey)) return;
            
            const chunkGroup = new THREE.Group();
            const startX = cx * chunkSize;
            const startZ = cz * chunkSize;

            for (let x = 0; x < chunkSize; x++) {
                for (let z = 0; z < chunkSize; z++) {
                    const worldX = startX + x;
                    const worldZ = startZ + z;
                    
                    const elevation = Math.floor(noise(worldX, worldZ)) + 12; 

                    // Bedrock sul fondo
                    addBlockToChunk(worldX, 0, worldZ, 3, chunkGroup);
                    
                    // Ottimizzazione RAM: genero solo gli ultimi 4-5 blocchi di superficie, non tutto il cubo
                    for(let y = Math.max(1, elevation - 4); y <= elevation; y++) {
                        if (y >= elevation - 1) {
                            addBlockToChunk(worldX, y, worldZ, 1, chunkGroup); 
                        } else {
                            addBlockToChunk(worldX, y, worldZ, 2, chunkGroup); 
                        }
                    }

                    // Alberi
                    if (Math.random() < 0.01 && elevation > 11) {
                        const treeHeight = Math.floor(Math.random() * 3) + 4;
                        for (let ty = 1; ty <= treeHeight; ty++) {
                            addBlockToChunk(worldX, elevation + ty, worldZ, 4, chunkGroup);
                        }
                        for (let lx = -1; lx <= 1; lx++) {
                            for (let lz = -1; lz <= 1; lz++) {
                                addBlockToChunk(worldX + lx, elevation + treeHeight, worldZ + lz, 7, chunkGroup);
                                addBlockToChunk(worldX + lx, elevation + treeHeight + 1, worldZ + lz, 7, chunkGroup);
                            }
                        }
                    }
                }
            }
            scene.add(chunkGroup);
            chunks.set(chunkKey, chunkGroup);
        }

        function addBlockToChunk(x, y, z, type, group) {
            const mesh = new THREE.Mesh(geometry, materials[type]);
            mesh.position.set(x, y, z);
            mesh.matrixAutoUpdate = false; 
            mesh.updateMatrix();
            group.add(mesh);
            blocks.set(getBlockKey(x, y, z), mesh);
        }

        // Accoda i chunk invece di crearli tutti di botto bloccando lo schermo
        function queueChunks() {
            const px = Math.floor(controls.getObject().position.x / chunkSize);
            const pz = Math.floor(controls.getObject().position.z / chunkSize);

            for (let x = -renderDistance; x <= renderDistance; x++) {
                for (let z = -renderDistance; z <= renderDistance; z++) {
                    const cx = px + x;
                    const cz = pz + z;
                    const chunkKey = getChunkKey(cx, cz);
                    
                    if (!chunks.has(chunkKey) && !chunkQueue.some(q => q.x === cx && q.z === cz)) {
                        chunkQueue.push({x: cx, z: cz});
                    }
                }
            }

            // Culling - Rimuove i chunk troppo distanti
            chunks.forEach((group, key) => {
                const [cx, cz] = key.split(',').map(Number);
                if (Math.abs(cx - px) > renderDistance + 1 || Math.abs(cz - pz) > renderDistance + 1) {
                    scene.remove(group);
                    chunks.delete(key);
                    group.children.forEach(mesh => {
                        blocks.delete(getBlockKey(mesh.position.x, mesh.position.y, mesh.position.z));
                    });
                }
            });
        }

        // 4. MECCANICHE E CONTROLLI (Inalterati)
        let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false;
        let canJump = false, isFlying = false;
        let velocity = new THREE.Vector3();
        let direction = new THREE.Vector3();
        let lastSpaceTime = 0;

        document.addEventListener('keydown', (event) => {
            if(document.getElementById('inventory').style.display === 'grid') return;

            switch (event.code) {
                case 'KeyW': moveForward = true; break;
                case 'KeyA': moveLeft = true; break;
                case 'KeyS': moveBackward = true; break;
                case 'KeyD': moveRight = true; break;
                case 'Space': 
                    const now = Date.now();
                    if (now - lastSpaceTime < 300) { 
                        isFlying = !isFlying;
                        velocity.y = 0;
                    } else if (canJump && !isFlying) {
                        velocity.y = 10;
                    } else if (isFlying) {
                        velocity.y = 10;
                    }
                    lastSpaceTime = now;
                    break;
                case 'ShiftLeft': if(isFlying) velocity.y = -10; break;
                case 'KeyE':
                    if (controls.isLocked) {
                        controls.unlock();
                        document.getElementById('inventory').style.display = 'grid';
                    }
                    break;
            }
        });

        document.addEventListener('keyup', (event) => {
            switch (event.code) {
                case 'KeyW': moveForward = false; break;
                case 'KeyA': moveLeft = false; break;
                case 'KeyS': moveBackward = false; break;
                case 'KeyD': moveRight = false; break;
                case 'Space': if(isFlying) velocity.y = 0; break;
                case 'ShiftLeft': if(isFlying) velocity.y = 0; break;
            }
        });

        document.addEventListener('mousedown', (e) => {
            if (document.getElementById('inventory').style.display === 'grid') {
                document.getElementById('inventory').style.display = 'none';
                controls.lock();
                return;
            }
            if (!controls.isLocked) { controls.lock(); return; }

            const raycaster = new THREE.Raycaster();
            raycaster.setFromCamera(new THREE.Vector2(0, 0), camera);
            
            const intersects = raycaster.intersectObjects(scene.children, true);
            if (intersects.length > 0 && intersects[0].distance < 6) {
                const intersect = intersects[0];
                const blockMesh = intersect.object;

                if (e.button === 0) { 
                    if (blockMesh.material !== materials[3]) { 
                        blockMesh.parent.remove(blockMesh);
                        blocks.delete(getBlockKey(blockMesh.position.x, blockMesh.position.y, blockMesh.position.z));
                    }
                } else if (e.button === 2) { 
                    const pos = intersect.point.clone().add(intersect.face.normal.clone().multiplyScalar(0.5));
                    const newPos = new THREE.Vector3(Math.round(pos.x), Math.round(pos.y), Math.round(pos.z));
                    
                    const pPos = controls.getObject().position;
                    if (Math.abs(newPos.x - pPos.x) < 0.8 && Math.abs(newPos.z - pPos.z) < 0.8 && newPos.y >= pPos.y - 1.5 && newPos.y <= pPos.y + 0.5) return; 

                    const newMesh = new THREE.Mesh(geometry, materials[currentBlockType]);
                    newMesh.position.copy(newPos);
                    newMesh.matrixAutoUpdate = false;
                    newMesh.updateMatrix();
                    
                    // Piazzalo nel chunk centrale temporaneamente o nella scena principale
                    scene.add(newMesh); 
                    blocks.set(getBlockKey(newPos.x, newPos.y, newPos.z), newMesh);
                }
            }
        });

        // 5. FISICA E COLLISIONI (Inalterate)
        function checkCollisions(pos) {
            const padding = 0.3; 
            const checkPoints = [
                new THREE.Vector3(pos.x - padding, pos.y, pos.z - padding),
                new THREE.Vector3(pos.x + padding, pos.y, pos.z + padding),
                new THREE.Vector3(pos.x - padding, pos.y, pos.z + padding),
                new THREE.Vector3(pos.x + padding, pos.y, pos.z - padding)
            ];

            for (let pt of checkPoints) {
                const key = getBlockKey(pt.x, pt.y - 1.5, pt.z); 
                if (blocks.has(key)) return true;
            }
            return false;
        }

        // 6. LOOP DI RENDERING
        let prevTime = performance.now();
        
        function animate() {
            requestAnimationFrame(animate);
            const time = performance.now();
            // Previene salti enormi di frame se il tab va in background
            const delta = Math.min((time - prevTime) / 1000, 0.1); 

            // Coda di generazione: genera 1 chunk al frame per non laggare il PC
            if (chunkQueue.length > 0) {
                const chunk = chunkQueue.shift();
                generateChunk(chunk.x, chunk.z);
            }

            if (controls.isLocked) {
                direction.z = Number(moveForward) - Number(moveBackward);
                direction.x = Number(moveRight) - Number(moveLeft);
                direction.normalize();

                const speed = isFlying ? 200 : 40;
                if (moveForward || moveBackward) velocity.z -= direction.z * speed * delta;
                if (moveLeft || moveRight) velocity.x -= direction.x * speed * delta;

                velocity.x -= velocity.x * 10.0 * delta;
                velocity.z -= velocity.z * 10.0 * delta;

                const pos = controls.getObject().position;
                if (!isFlying) {
                    velocity.y -= 25.0 * delta; 
                    const isGrounded = checkCollisions(pos);
                    
                    if (isGrounded && velocity.y < 0) {
                        velocity.y = 0;
                        canJump = true;
                        // Regolazione collisione
                        pos.y = Math.floor(pos.y - 1.5) + 2.0; 
                    } else {
                        canJump = false;
                    }
                } else {
                    velocity.y -= velocity.y * 10.0 * delta; 
                }

                controls.moveRight(-velocity.x * delta);
                controls.moveForward(-velocity.z * delta);
                controls.getObject().position.y += velocity.y * delta;
                
                if (controls.getObject().position.y < -10) controls.getObject().position.y = 25;

                queueChunks(); // Controlla costantemente se servono nuovi chunk
            }

            renderer.render(scene, camera);
            prevTime = time;
        }

        // 7. AVVIO IMMEDIATO
        controls.getObject().position.set(0, 25, 0); 
        
        // Costruisce istantaneamente il terreno sotto i tuoi piedi per non farti cadere nel vuoto
        generateChunk(0, 0);
        
        // Dissolve la schermata di caricamento immediatamente
        const loadingScreen = document.getElementById('loading');
        loadingScreen.style.opacity = '0';
        setTimeout(() => loadingScreen.style.display = 'none', 500);

        animate();

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

    </script>
</body>
</html>

