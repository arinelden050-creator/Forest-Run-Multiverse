<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Forest Run: Ultimate Fusion</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@900&display=swap');
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; touch-action: none; }
        #ui { position: absolute; top: 20px; width: 100%; display: flex; justify-content: space-around; color: #0f0; z-index: 100; text-shadow: 0 0 10px #0f0; }
        #health-bar, #boss-ui { width: 150px; height: 12px; background: #222; border: 2px solid #0f0; border-radius: 5px; overflow: hidden; }
        #health-fill { width: 100%; height: 100%; background: #0f0; transition: 0.2s; }
        #boss-container { position: absolute; top: 70px; left: 50%; transform: translateX(-50%); display: none; text-align: center; color: #f00; }
        #boss-fill { width: 100%; height: 100%; background: #f00; }
        #msg { position: absolute; top: 50%; width: 100%; text-align: center; color: #f00; font-size: 30px; display: none; z-index: 200; }
        #flash { position: absolute; inset: 0; background: white; opacity: 0; pointer-events: none; z-index: 1000; transition: opacity 0.15s; }
        canvas { display: block; }
    </style>
</head>
<body>

    <div id="flash"></div>
    <div id="ui">
        <div>PUAN: <span id="score">0</span></div>
        <div id="world-name">ORMAN</div>
        <div>CAN: <div id="health-bar"><div id="health-fill"></div></div></div>
    </div>

    <div id="boss-container">
        <div>GIGA-TREE BOSS</div>
        <div id="boss-ui" style="border-color: #f00; width: 300px;"><div id="boss-fill"></div></div>
    </div>

    <div id="msg">SİSTEM ÇÖKTÜ!</div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

    <script>
        let scene, camera, renderer, player, ground, stars, boss;
        let obstacles = [], bullets = [], particles = [];
        let score = 0, health = 100, active = true, speed = 0.8;
        let targetX = 0, isJetpack = false, gravityDirection = 1;
        let currentWorld = "forest", bossActive = false, bossHealth = 100;
        const skyLimit = 12;

        function init() {
            scene = new THREE.Scene();
            setForestWorld();

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 5, 12);
            
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            scene.add(new THREE.AmbientLight(0xffffff, 0.8));
            const sun = new THREE.DirectionalLight(0xffffff, 1);
            sun.position.set(5, 10, 5);
            scene.add(sun);

            createStars();
            createPlayer();
            setupControls();
            animate();
        }

        function setForestWorld() {
            currentWorld = "forest";
            scene.background = new THREE.Color(0x000500);
            scene.fog = new THREE.Fog(0x000500, 10, 75);
            updateUI("#0f0", "ORMAN");
            replaceGround(0x051a05, false);
        }

        function setCyberWorld() {
            currentWorld = "cyber";
            scene.background = new THREE.Color(0x00001a);
            scene.fog = new THREE.Fog(0x00001a, 10, 80);
            updateUI("#0ff", "SİBER");
            replaceGround(0x000022, true);
        }

        function updateUI(color, name) {
            const el = document.getElementById('world-name');
            el.innerText = name;
            el.style.color = color;
        }

        function replaceGround(color, wireframe) {
            if(ground) scene.remove(ground);
            ground = new THREE.Mesh(
                new THREE.PlaneGeometry(25, 1000, 10, 50), 
                new THREE.MeshPhongMaterial({ color: color, wireframe: wireframe })
            );
            ground.rotation.x = -Math.PI / 2;
            scene.add(ground);
        }

        function createPlayer() {
            player = new THREE.Group();
            const body = new THREE.Mesh(new THREE.BoxGeometry(1, 1.8, 0.8), new THREE.MeshPhongMaterial({color: 0x00ff00}));
            body.position.y = 1.5;
            const head = new THREE.Mesh(new THREE.BoxGeometry(0.7, 0.7, 0.7), new THREE.MeshPhongMaterial({color: 0xffffff}));
            head.position.y = 2.8;
            const jet = new THREE.Mesh(new THREE.BoxGeometry(0.8, 1.2, 0.5), new THREE.MeshPhongMaterial({color: 0x333333}));
            jet.position.set(0, 1.5, -0.6);
            player.add(body, head, jet);
            scene.add(player);
        }

        function createBoss() {
            bossActive = true;
            bossHealth = 100;
            document.getElementById('boss-container').style.display = "block";
            document.getElementById('boss-fill').style.width = "100%";
            
            boss = new THREE.Group();
            const trunk = new THREE.Mesh(new THREE.CylinderGeometry(2, 4, 15), new THREE.MeshPhongMaterial({color: 0x222222, wireframe: true}));
            const eye = new THREE.Mesh(new THREE.SphereGeometry(3, 16, 16), new THREE.MeshPhongMaterial({color: 0xff0000, emissive: 0xff0000}));
            eye.position.y = 10;
            boss.add(trunk, eye);
            boss.position.set(0, 0, -80);
            scene.add(boss);
        }

        function shoot() {
            const beam = new THREE.Mesh(
                new THREE.CylinderGeometry(0.06, 0.06, 4), 
                new THREE.MeshBasicMaterial({ color: 0x00ffff, transparent: true, opacity: 0.8 })
            );
            beam.position.set(player.position.x, player.position.y + 1.5, player.position.z - 2);
            beam.rotation.x = Math.PI / 2;
            scene.add(beam);
            bullets.push(beam);
        }

        function spawnObstacle() {
            if(!active || bossActive) return;
            let rand = Math.random();

            // Evrensel Kaos (Meteor)
            if(rand < 0.02) {
                const met = new THREE.Mesh(new THREE.DodecahedronGeometry(1.2), new THREE.MeshPhongMaterial({color: 0xff5500, emissive: 0x330000}));
                met.position.set((Math.random()-0.5)*20, 20, -100);
                met.userData = { isMeteor: true };
                scene.add(met);
                obstacles.push(met);
            }

            // Dünyaya Özel Engeller
            if(currentWorld === "forest") {
                if(rand < 0.06) spawnTree();
                if(rand < 0.015) spawnGate(); // Yerçekimi kapısı
            } else {
                if(rand < 0.08) spawnCyberBlock();
            }

            // Portal (Boyut Değiştirme) - Boss yokken çıkar
            if(rand < 0.008 && score > 1000) {
                const portal = new THREE.Mesh(new THREE.SphereGeometry(3.5, 32, 32), new THREE.MeshPhongMaterial({ color: 0xffffff, wireframe: true }));
                portal.position.set((Math.random()-0.5)*10, 5, -120);
                portal.userData = { isPortal: true };
                scene.add(portal);
                obstacles.push(portal);
            }
        }

        function spawnTree() {
            const tree = new THREE.Mesh(new THREE.BoxGeometry(2, 6, 2), new THREE.MeshPhongMaterial({color: 0x004400}));
            tree.position.set((Math.random()-0.5)*20, 3, -100);
            scene.add(tree);
            obstacles.push(tree);
        }

        function spawnCyberBlock() {
            const obs = new THREE.Mesh(new THREE.BoxGeometry(2, 10, 2), new THREE.MeshPhongMaterial({ color: 0xff00ff, emissive: 0x220022 }));
            obs.position.set((Math.random()-0.5)*20, 5, -100);
            scene.add(obs);
            obstacles.push(obs);
        }

        function spawnGate() {
            const gate = new THREE.Mesh(new THREE.TorusGeometry(3, 0.3, 16, 32), new THREE.MeshPhongMaterial({ color: 0x00ffff, emissive: 0x004444 }));
            gate.position.set((Math.random()-0.5)*12, 5, -100);
            gate.userData = { isGate: true };
            scene.add(gate);
            obstacles.push(gate);
        }

        function animate() {
            if(!active) return;
            requestAnimationFrame(animate);

            // Oyuncu Hareketi
            player.position.x += (targetX - player.position.x) * 0.1;
            player.position.y += (isJetpack ? 0.32 : -0.22) * gravityDirection;
            if (player.position.y < 0) player.position.y = 0;
            if (player.position.y > skyLimit) player.position.y = skyLimit;
            player.rotation.z = THREE.MathUtils.lerp(player.rotation.z, (gravityDirection === -1 ? Math.PI : 0), 0.1);

            // Boss Mantığı
            if (bossActive) {
                boss.position.x = Math.sin(Date.now() * 0.002) * 9;
                boss.position.z = THREE.MathUtils.lerp(boss.position.z, -35, 0.015);
                if (Math.random() < 0.06) {
                    const atk = new THREE.Mesh(new THREE.SphereGeometry(1.2), new THREE.MeshBasicMaterial({color: 0xff0000}));
                    atk.position.set(boss.position.x, 10, boss.position.z);
                    atk.userData = { isAttack: true };
                    scene.add(atk);
                    obstacles.push(atk);
                }
            } else if (Math.floor(score/10) > 0 && Math.floor(score/10) % 5000 === 0) {
                if(!bossActive) createBoss();
            }

            // Mermiler
            bullets.forEach((b, i) => {
                b.position.z -= 4.5;
                if(bossActive && b.position.distanceTo(boss.position) < 8) {
                    bossHealth -= 2.5;
                    document.getElementById('boss-fill').style.width = bossHealth + "%";
                    explode(b.position, 0xff0000);
                    scene.remove(b); bullets.splice(i, 1);
                    if(bossHealth <= 0) {
                        bossActive = false;
                        scene.remove(boss);
                        document.getElementById('boss-container').style.display = "none";
                        score += 50000;
                        teleport();
                    }
                }
                if(b.position.z < -150) { scene.remove(b); bullets.splice(i, 1); }
            });

            // Engeller ve Çarpışmalar
            obstacles.forEach((t, i) => {
                t.position.z += (t.userData.isAttack ? 1.6 : speed);
                if(t.userData.isMeteor) { t.position.y -= 0.25; t.rotation.x += 0.1; }

                // Özel Temaslar
                let dist = player.position.distanceTo(t.position);
                if(t.userData.isPortal && dist < 5) teleport();
                if(t.userData.isGate && dist < 4) {
                    gravityDirection *= -1;
                    scene.remove(t); obstacles.splice(i, 1);
                    return;
                }

                // Ölümcül Çarpışma
                if(!t.userData.isPortal && !t.userData.isGate && dist < 2.2) {
                    health -= 20;
                    document.getElementById('health-fill').style.width = health + "%";
                    explode(t.position, 0x00ff00);
                    scene.remove(t); obstacles.splice(i, i);
                    if(health <= 0) endGame();
                }

                if(t.position.z > 20) { scene.remove(t); obstacles.splice(i, 1); }
            });

            // Parçacıklar
            particles.forEach((p, i) => {
                p.position.add(p.userData.v);
                p.scale.multiplyScalar(0.95);
                if(p.scale.x < 0.01) { scene.remove(p); particles.splice(i, 1); }
            });

            spawnObstacle();
            score += 10;
            speed += 0.0001;
            document.getElementById('score').innerText = Math.floor(score/10);
            renderer.render(scene, camera);
        }

        function setupControls() {
            window.addEventListener('touchstart', () => { isJetpack = true; shoot(); });
            window.addEventListener('touchend', () => { isJetpack = false; });
            window.addEventListener('touchmove', (e) => { 
                targetX = ((e.touches[0].clientX / window.innerWidth) - 0.5) * 22; 
            });
        }

        function createStars() {
            const geo = new THREE.BufferGeometry();
            const pos = [];
            for(let i=0; i<1500; i++) pos.push((Math.random()-0.5)*400, (Math.random()-0.5)*400, (Math.random()-0.5)*400);
            geo.setAttribute('position', new THREE.Float32BufferAttribute(pos, 3));
            stars = new THREE.Points(geo, new THREE.PointsMaterial({color: 0xffffff, size: 0.6}));
            scene.add(stars);
        }

        function explode(pos, color) {
            for(let i=0; i<6; i++) {
                const p = new THREE.Mesh(new THREE.BoxGeometry(0.4,0.4,0.4), new THREE.MeshBasicMaterial({color: color}));
                p.position.copy(pos);
                p.userData = { v: new THREE.Vector3((Math.random()-0.5)*0.6, (Math.random()-0.5)*0.6, (Math.random()-0.5)*0.6) };
                scene.add(p);
                particles.push(p);
            }
        }

        function teleport() {
            const f = document.getElementById('flash'); f.style.opacity = 1;
            setTimeout(() => f.style.opacity = 0, 150);
            obstacles.forEach(o => scene.remove(o)); obstacles = [];
            currentWorld === "forest" ? setCyberWorld() : setForestWorld();
            player.position.y = 2;
            gravityDirection = 1;
        }

        function endGame() { 
            active = false; 
            document.getElementById('msg').style.display = "block"; 
            setTimeout(() => location.reload(), 3000); 
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        init();
    </script>
</body>
</html>
