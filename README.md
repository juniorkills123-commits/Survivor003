<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Archero Mobile - Fase 1</title>
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
            margin: 0;
            padding: 0;
        }
        body {
            background-color: #111;
            color: #fff;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
            display: flex;
            align-items: center;
            justify-content: center;
            height: 100vh;
        }
        #game-container {
            position: relative;
            width: 100vw;
            height: 100vh;
            max-width: 500px;
            max-height: 850px;
            background-color: #1e2e1c;
            overflow: hidden;
            box-shadow: 0 0 30px rgba(0,0,0,0.8);
        }
        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }
        #hud {
            position: absolute;
            top: 20px;
            left: 20px;
            right: 20px;
            pointer-events: none;
            font-size: 16px;
            font-weight: bold;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.9);
            z-index: 10;
            display: flex;
            justify-content: space-between;
        }
        .hp-container {
            display: flex;
            align-items: center;
            gap: 8px;
        }
        .bar-bg {
            width: 120px;
            height: 16px;
            background-color: #444;
            border: 2px solid #fff;
            border-radius: 8px;
            overflow: hidden;
        }
        #hp-bar {
            width: 100%;
            height: 100%;
            background-color: #ff3333;
            transition: width 0.1s;
        }
        #joystick-container {
            position: absolute;
            bottom: 50px;
            left: 0;
            right: 0;
            height: 180px;
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 10;
        }
        #joystick-base {
            width: 110px;
            height: 110px;
            background-color: rgba(255, 255, 255, 0.1);
            border: 3px solid rgba(255, 255, 255, 0.3);
            border-radius: 50%;
            position: relative;
            backdrop-filter: blur(4px);
        }
        #joystick-stick {
            width: 44px;
            height: 44px;
            background-color: rgba(255, 255, 255, 0.6);
            border-radius: 50%;
            position: absolute;
            top: 30px;
            left: 30px;
            box-shadow: 0 4px 10px rgba(0,0,0,0.5);
        }
        .screen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(10, 15, 10, 0.95);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 20;
            text-align: center;
            padding: 30px;
        }
        .btn {
            padding: 15px 40px;
            background-color: #ffcc00;
            color: #000;
            border: none;
            border-radius: 25px;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
            margin-top: 20px;
            box-shadow: 0 4px 15px rgba(255,204,0,0.3);
        }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="hud">
            <div class="hp-container">
                <span>HP</span>
                <div class="bar-bg"><div id="hp-bar"></div></div>
            </div>
            <div id="score-txt">Abates: 0</div>
        </div>

        <canvas id="gameCanvas"></canvas>

        <div id="joystick-container">
            <div id="joystick-base">
                <div id="joystick-stick"></div>
            </div>
        </div>

        <div id="overScreen" class="screen" style="display:none;">
            <h1 style="color: #ff3333; font-size: 32px; margin-bottom: 10px;">DERROTADO</h1>
            <p id="over-desc">Você caiu na masmorra.</p>
            <button class="btn" onclick="resetGame()">TENTAR DE NOVO</button>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const hpBar = document.getElementById('hp-bar');
        const scoreTxt = document.getElementById('score-txt');
        const overScreen = document.getElementById('overScreen');
        const overDesc = document.getElementById('over-desc');

        function resize() {
            const container = document.getElementById('game-container');
            canvas.width = container.clientWidth;
            canvas.height = container.clientHeight;
        }
        window.addEventListener('resize', resize);
        resize();

        // Configurações e variáveis do jogo
        let player = { x: 0, y: 0, radius: 16, speed: 3.5, hp: 100, maxHp: 100, isMoving: false, shootCooldown: 0, shootInterval: 350 };
        let enemies = [];
        let bullets = [];
        let score = 0;
        let gameActive = true;
        let lastTime = 0;
        let spawnTimer = 0;
        let spawnInterval = 1500;

        // Joystick Virtual
        const joyContainer = document.getElementById('joystick-container');
        const joyBase = document.getElementById('joystick-base');
        const joyStick = document.getElementById('joystick-stick');
        let joyActive = false, joyStart = { x: 0, y: 0 }, moveVec = { x: 0, y: 0 };

        joyContainer.addEventListener('touchstart', (e) => {
            joyActive = true;
            const rect = joyBase.getBoundingClientRect();
            joyStart.x = rect.left + rect.width / 2;
            joyStart.y = rect.top + rect.height / 2;
            updateJoystick(e.touches[0].clientX, e.touches[0].clientY);
        });

        joyContainer.addEventListener('touchmove', (e) => {
            if (!joyActive) return;
            updateJoystick(e.touches[0].clientX, e.touches[0].clientY);
        });

        joyContainer.addEventListener('touchend', () => {
            joyActive = false;
            joyStick.style.left = '30px';
            joyStick.style.top = '30px';
            moveVec = { x: 0, y: 0 };
            player.isMoving = false;
        });

        function updateJoystick(cx, cy) {
            let dx = cx - joyStart.x;
            let dy = cy - joyStart.y;
            let dist = Math.sqrt(dx * dx + dy * dy);
            let maxRadius = 35;

            if (dist > maxRadius) {
                dx = (dx / dist) * maxRadius;
                dy = (dy / dist) * maxRadius;
                dist = maxRadius;
            }

            joyStick.style.left = `${30 + dx}px`;
            joyStick.style.top = `${30 + dy}px`;

            if (dist > 5) {
                moveVec.x = dx / maxRadius;
                moveVec.y = dy / maxRadius;
                player.isMoving = true;
            } else {
                moveVec.x = 0;
                moveVec.y = 0;
                player.isMoving = false;
            }
        }

        function spawnEnemy() {
            let edge = Math.floor(Math.random() * 4);
            let x, y;
            if (edge === 0) { x = Math.random() * canvas.width; y = -20; }
            else if (edge === 1) { x = canvas.width + 20; y = Math.random() * canvas.height; }
            else if (edge === 2) { x = Math.random() * canvas.width; y = canvas.height + 20; }
            else { x = -20; y = Math.random() * canvas.height; }

            enemies.push({ x: x, y: y, radius: 12, speed: 1.2 + Math.min(score * 0.03, 1.5), hp: 1 });
        }

        function resetGame() {
            player.x = canvas.width / 2;
            player.y = canvas.height * 0.7;
            player.hp = player.maxHp;
            player.isMoving = false;
            enemies = [];
            bullets = [];
            score = 0;
            spawnInterval = 1500;
            scoreTxt.innerText = `Abates: ${score}`;
            hpBar.style.width = '100%';
            overScreen.style.display = 'none';
            gameActive = true;
            lastTime = performance.now();
            requestAnimationFrame(loop);
        }

        function loop(now) {
            if (!gameActive) return;
            let dt = now - lastTime;
            lastTime = now;

            // 1. Movimentação do Jogador
            if (player.isMoving) {
                player.x += moveVec.x * player.speed;
                player.y += moveVec.y * player.speed;
                player.x = Math.max(player.radius, Math.min(canvas.width - player.radius, player.x));
                player.y = Math.max(player.radius, Math.min(canvas.height - player.radius, player.y));
            } else {
                // 2. Ataque Automático (Apenas se estiver PARADO)
                player.shootCooldown -= dt;
                if (player.shootCooldown <= 0 && enemies.length > 0) {
                    let closest = null, minDist = Infinity;
                    enemies.forEach(e => {
                        let d = Math.sqrt((e.x - player.x)**2 + (e.y - player.y)**2);
                        if (d < minDist) { minDist = d; closest = e; }
                    });

                    if (closest) {
                        let angle = Math.atan2(closest.y - player.y, closest.x - player.x);
                        bullets.push({
                            x: player.x, y: player.y,
                            vx: Math.cos(angle) * 8, vy: Math.sin(angle) * 8,
                            radius: 4
                        });
                        player.shootCooldown = player.shootInterval;
                    }
                }
            }

            // Spawner de inimigos
            spawnTimer += dt;
            if (spawnTimer >= spawnInterval) {
                spawnEnemy();
                spawnTimer = 0;
                spawnInterval = Math.max(600, 1500 - score * 20);
            }

            // Atualizar Projéteis
            for (let i = bullets.length - 1; i >= 0; i--) {
                let b = bullets[i];
                b.x += b.vx; b.y += b.vy;
                if (b.x < 0 || b.x > canvas.width || b.y < 0 || b.y > canvas.height) {
                    bullets.splice(i, 1);
                }
            }

            // Atualizar Inimigos e Colisões
            for (let i = enemies.length - 1; i >= 0; i--) {
                let e = enemies[i];
                let dx = player.x - e.x, dy = player.y - e.y;
                let dist = Math.sqrt(dx*dx + dy*dy);

                // IA: Seguir jogador
                e.x += (dx / dist) * e.speed;
                e.y += (dy / dist) * e.speed;

                // Colisão Inimigo com Jogador
                if (dist < player.radius + e.radius) {
                    player.hp -= 20;
                    hpBar.style.width = `${Math.max(0, (player.hp / player.maxHp) * 100)}%`;
                    enemies.splice(i, 1);
                    if (player.hp <= 0) {
                        gameActive = false;
                        overDesc.innerText = `Você eliminou ${score} monstros antes de cair.`;
                        overScreen.style.display = 'flex';
                        return;
                    }
                    continue;
                }

                // Colisão Bala com Inimigo
                for (let j = bullets.length - 1; j >= 0; j--) {
                    let b = bullets[j];
                    let bDist = Math.sqrt((e.x - b.x)**2 + (e.y - b.y)**2);
                    if (bDist < e.radius + b.radius) {
                        enemies.splice(i, 1);
                        bullets.splice(j, 1);
                        score++;
                        scoreTxt.innerText = `Abates: ${score}`;
                        break;
                    }
                }
            }

            // Renderização Gráfica
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Chão da Masmorra
            ctx.fillStyle = '#22331f';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Desenhar Projéteis
            ctx.fillStyle = '#ffcc00';
            bullets.forEach(b => {
                ctx.beginPath(); ctx.arc(b.x, b.y, b.radius, 0, Math.PI*2); ctx.fill();
            });

            // Desenhar Inimigos (Monstros Vermelhos)
            ctx.fillStyle = '#e74c3c';
            enemies.forEach(e => {
                ctx.beginPath(); ctx.arc(e.x, e.y, e.radius, 0, Math.PI*2); ctx.fill();
            });

            // Desenhar Jogador (Arqueiro Azul)
            ctx.fillStyle = '#3498db';
            ctx.beginPath(); ctx.arc(player.x, player.y, player.radius, 0, Math.PI*2); ctx.fill();

            // Anel indicador de mira se estiver parado
            if (!player.isMoving) {
                ctx.strokeStyle = 'rgba(255, 255, 255, 0.3)';
                ctx.lineWidth = 2;
                ctx.beginPath(); ctx.arc(player.x, player.y, player.radius + 6, 0, Math.PI*2); ctx.stroke();
            }

            requestAnimationFrame(loop);
        }

        // Início automático
        resetGame();
    </script>
</body>
</html>
