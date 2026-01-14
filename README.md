<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>충주맨의 항아리 등반 (지렛대 에디션)</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #87CEEB;
            font-family: 'Noto Sans KR', sans-serif;
            user-select: none;
            touch-action: none; /* 터치 스크롤 방지 */
            cursor: crosshair; /* 정밀 조작 커서 */
        }
        #gameCanvas {
            display: block;
        }
        #ui-layer {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(255, 255, 255, 0.9);
            padding: 15px;
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.2);
            text-align: right;
            border: 2px solid #333;
            pointer-events: auto; /* UI는 클릭 가능 */
        }
        button {
            font-family: inherit;
            background: #FF6347;
            color: white;
            border: none;
            padding: 8px 15px;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
            transition: background 0.2s;
        }
        button:hover {
            background: #FF4500;
        }
        h1 {
            position: absolute;
            top: 20px;
            left: 20px;
            margin: 0;
            color: #222;
            text-shadow: 2px 2px 0px white;
            font-size: 2rem;
            pointer-events: none;
            background: rgba(255,255,255,0.7);
            padding: 5px 15px;
            border-radius: 10px;
        }
        .location-indicator {
            position: absolute;
            top: 80px;
            left: 20px;
            font-size: 1.5rem;
            font-weight: bold;
            color: #fff;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
            pointer-events: none;
        }
        #game-over-msg {
            display: none;
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            font-size: 4rem;
            font-weight: 900;
            color: #ff0000;
            text-shadow: 
                3px 3px 0 #fff, 
                -3px -3px 0 #fff, 
                3px -3px 0 #fff, 
                -3px 3px 0 #fff,
                0px 0px 20px rgba(0,0,0,0.5);
            text-align: center;
            z-index: 100;
            white-space: nowrap;
            animation: popIn 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            pointer-events: none;
        }
        @keyframes popIn {
            from { transform: translate(-50%, -50%) scale(0.5); opacity: 0; }
            to { transform: translate(-50%, -50%) scale(1); opacity: 1; }
        }
    </style>
</head>
<body>

    <h1>충주맨의 충주 여행</h1>
    <div id="locationDisplay" class="location-indicator">현재 위치: 출발지</div>
    
    <div id="game-over-msg">충주맨은 해고 되었습니다</div>

    <div id="ui-layer">
        <div style="margin-bottom: 10px;"><strong>충주맨 설정</strong></div>
        <label for="charInput" style="cursor: pointer; display: inline-block; background: #4CAF50; color: white; padding: 8px 12px; border-radius: 5px; font-size: 14px; margin-bottom: 5px;">
            📸 사진 변경하기
        </label>
        <input type="file" id="charInput" accept="image/*" style="display: none;">
        <div style="font-size: 11px; color: #666; margin-bottom: 10px;">
            *기본: 충주맨 망치 (2).png
        </div>
        <button onclick="resetGame()">🔄 처음부터 다시</button>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const locDisplay = document.getElementById('locationDisplay');
        const gameOverMsg = document.getElementById('game-over-msg');

        // === 게임 설정 ===
        let width, height;
        const gravity = 0.8; 
        const airDrag = 0.99; 
        
        let cameraX = 0;
        let cameraY = 0;
        let isGameOver = false;

        // 플레이어
        const player = {
            x: 200,
            y: -300, 
            vx: 0,
            vy: 0,
            radius: 50,
            angle: 0,   
            prevAngle: 0 
        };

        // 망치
        const hammer = {
            length: 150, 
            headRadius: 30, 
            x: 0,
            y: 0,
            friction: 0.9 
        };

        // 물리 재질
        const materials = {
            0: { friction: 0.7, bounce: 0.1, name: "바위/흙" },
            1: { friction: 0.1, bounce: 0.05, name: "철/얼음" },
            2: { friction: 0.9, bounce: 0.1, name: "나무" }
        };

        // 맵 데이터
        const platforms = [
            // 공중 발판들
            { x: 200, y: -40, w: 60, h: 40, type: 2 },
            { x: 300, y: -80, w: 80, h: 50, type: 2 },
            { x: 400, y: -120, w: 60, h: 30, type: 2 },
            { x: 500, y: -160, w: 80, h: 50, type: 2 },
            { x: 600, y: -200, w: 60, h: 30, type: 2 },
            { x: 700, y: -150, w: 100, h: 30, type: 2 },
            { x: 850, y: -120, w: 50, h: 20, type: 2 },
            { x: 1000, y: -100, w: 200, h: 300, type: 0 },
            
            // 사과 농장
            { x: 1200, y: -50, w: 500, h: 50, type: 2 },
            { x: 1250, y: -120, w: 60, h: 20, type: 2 },
            { x: 1250, y: -180, w: 80, h: 20, type: 2 },
            { x: 1350, y: -220, w: 60, h: 20, type: 2 },
            { x: 1450, y: -260, w: 80, h: 20, type: 2 },
            { x: 1600, y: -250, w: 50, h: 400, type: 2 },
            { x: 1550, y: -300, w: 50, h: 20, type: 2 },
            { x: 1650, y: -350, w: 40, h: 20, type: 2 },
            { x: 1650, y: -450, w: 60, h: 20, type: 2 },
            { x: 1400, y: -350, w: 300, h: 40, type: 2 },
            { x: 1300, y: -450, w: 100, h: 20, type: 2 },
            
            // 징검다리
            { x: 1750, y: -200, w: 60, h: 20, type: 0 },
            { x: 1800, y: -250, w: 80, h: 30, type: 0 },
            { x: 1900, y: -200, w: 150, h: 50, type: 0 },
            { x: 2000, y: -240, w: 60, h: 20, type: 0 },
            { x: 2050, y: -280, w: 80, h: 30, type: 0 },
            { x: 2150, y: -290, w: 60, h: 20, type: 0 },
            { x: 2200, y: -300, w: 150, h: 50, type: 0 },
            { x: 2350, y: -250, w: 100, h: 30, type: 0 },
            { x: 2450, y: -270, w: 60, h: 20, type: 0 },
            { x: 2500, y: -250, w: 400, h: 50, type: 1 },
            
            // 절벽
            { x: 3000, y: -400, w: 1000, h: 500, type: 0 },
            { x: 3020, y: -450, w: 40, h: 20, type: 0 },
            { x: 3050, y: -500, w: 50, h: 20, type: 0 },
            { x: 3100, y: -530, w: 40, h: 20, type: 0 },
            { x: 3150, y: -550, w: 50, h: 20, type: 0 },
            { x: 3200, y: -600, w: 300, h: 30, type: 0 },
            { x: 3250, y: -650, w: 40, h: 20, type: 0 },
            { x: 3300, y: -700, w: 40, h: 20, type: 0 },
            { x: 3380, y: -720, w: 40, h: 20, type: 0 },
            { x: 3450, y: -750, w: 40, h: 20, type: 0 },
            { x: 3600, y: -800, w: 200, h: 30, type: 0 },
            { x: 3400, y: -1000, w: 50, h: 300, type: 0 },
            { x: 3350, y: -900, w: 50, h: 20, type: 0 },
            { x: 3450, y: -930, w: 30, h: 20, type: 0 },
            { x: 3420, y: -980, w: 30, h: 20, type: 0 },
            { x: 3500, y: -1200, w: 400, h: 50, type: 0 },
            
            // 중앙탑
            { x: 4100, y: -1200, w: 800, h: 200, type: 0 },
            { x: 4250, y: -1250, w: 100, h: 30, type: 0 },
            { x: 4350, y: -1320, w: 80, h: 30, type: 0 },
            { x: 4400, y: -1400, w: 300, h: 40, type: 0 },
            { x: 4420, y: -1480, w: 40, h: 20, type: 0 },
            { x: 4450, y: -1550, w: 200, h: 150, type: 0 },
            { x: 4500, y: -1700, w: 100, h: 150, type: 0 },
        ];

        let mouseX = 0;
        let mouseY = 0;
        
        const charImage = new Image();
        let imageLoaded = false;
        charImage.src = "충주맨 망치 (2).png"; 
        charImage.onload = () => { imageLoaded = true; };
        
        document.getElementById('charInput').addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = function(event) {
                    charImage.src = event.target.result;
                    imageLoaded = true;
                }
                reader.readAsDataURL(file);
            }
        });

        function resize() {
            width = canvas.width = window.innerWidth;
            height = canvas.height = window.innerHeight;
            if(player.y === 0) resetGame();
        }
        window.addEventListener('resize', resize);
        
        // 마우스 및 터치 이벤트 통합 처리
        function updateInput(x, y) {
            mouseX = x;
            mouseY = y;
        }

        window.addEventListener('mousemove', e => {
            updateInput(e.clientX, e.clientY);
        });

        // 터치 지원 추가
        window.addEventListener('touchstart', e => {
            if(e.touches.length > 0) {
                updateInput(e.touches[0].clientX, e.touches[0].clientY);
            }
        }, {passive: false});

        window.addEventListener('touchmove', e => {
            if(e.touches.length > 0) {
                e.preventDefault(); // 스크롤 방지
                updateInput(e.touches[0].clientX, e.touches[0].clientY);
            }
        }, {passive: false});


        function resetGame() {
            isGameOver = false;
            gameOverMsg.style.display = 'none';
            player.x = 220;
            player.y = -300; 
            player.vx = 0;
            player.vy = 0;
            cameraX = 0;
            cameraY = 0;
            player.prevAngle = 0;
            
            // 초기 마우스 위치를 플레이어 근처로 설정
            mouseX = width / 2;
            mouseY = height / 2;
        }

        function gameOver() {
            if(isGameOver) return;
            isGameOver = true;
            gameOverMsg.style.display = 'block';
            setTimeout(() => { resetGame(); }, 2500);
        }

        function checkCollision(x, y, r) {
            let collision = { hit: false, nx: 0, ny: 0, overlap: 0, type: 0 };
            
            for (let p of platforms) {
                if (x + r < p.x || x - r > p.x + p.w || y + r < p.y || y - r > p.y + p.h) continue;
                let closestX = Math.max(p.x, Math.min(x, p.x + p.w));
                let closestY = Math.max(p.y, Math.min(y, p.y + p.h));
                let dx = x - closestX;
                let dy = y - closestY;
                let distSq = dx * dx + dy * dy;
                if (distSq < r * r) {
                    let dist = Math.sqrt(distSq);
                    let overlap = r - dist;
                    let nx, ny;
                    if (dist === 0) { 
                         let dL = Math.abs(x - p.x);
                         let dR = Math.abs(x - (p.x + p.w));
                         let dT = Math.abs(y - p.y);
                         let dB = Math.abs(y - (p.y + p.h));
                         let min = Math.min(dL, dR, dT, dB);
                         if(min == dT) { nx=0; ny=-1; overlap = r + dT; }
                         else if(min == dB) { nx=0; ny=1; overlap = r + dB; }
                         else if(min == dL) { nx=-1; ny=0; overlap = r + dL; }
                         else { nx=1; ny=0; overlap = r + dR; }
                    } else {
                        nx = dx / dist; ny = dy / dist;
                    }
                    if (!collision.hit || overlap > collision.overlap) {
                        collision.hit = true; collision.overlap = overlap;
                        collision.nx = nx; collision.ny = ny; collision.type = p.type;
                    }
                }
            }
            return collision;
        }

        function update() {
            if(isGameOver) return;

            player.vy += gravity;
            player.vx *= airDrag;
            player.vy *= airDrag;

            if (player.y > 400) gameOver();

            let screenPx = player.x - cameraX;
            let screenPy = player.y - cameraY;
            let mouseDx = mouseX - screenPx; 
            let mouseDy = mouseY - screenPy;
            let targetAngle = Math.atan2(mouseDy, mouseDx);

            let angleDiff = targetAngle - player.prevAngle;
            while (angleDiff > Math.PI) angleDiff -= Math.PI * 2;
            while (angleDiff < -Math.PI) angleDiff += Math.PI * 2;
            
            let angularVelocity = angleDiff; 
            player.angle = targetAngle;
            player.prevAngle = player.angle;

            let idealHx = player.x + Math.cos(player.angle) * hammer.length;
            let idealHy = player.y + Math.sin(player.angle) * hammer.length;
            
            // 망치 머리의 목표 속도 (플레이어 속도 + 회전 속도)
            let hammerTanX = -Math.sin(player.angle);
            let hammerTanY = Math.cos(player.angle);
            let hammerVelX = player.vx + hammerTanX * angularVelocity * hammer.length;
            let hammerVelY = player.vy + hammerTanY * angularVelocity * hammer.length;

            let hammerCol = checkCollision(idealHx, idealHy, hammer.headRadius);

            if (hammerCol.hit) {
                // 망치 위치 확정 (벽 밖으로)
                hammer.x = idealHx + hammerCol.nx * hammerCol.overlap;
                hammer.y = idealHy + hammerCol.ny * hammerCol.overlap;

                let velAlongNormal = hammerVelX * hammerCol.nx + hammerVelY * hammerCol.ny;

                if (velAlongNormal < 0 || hammerCol.overlap > 0) {
                    
                    const PUSH_POWER = 0.08; 
                    const SWING_POWER = 0.15; // 튕겨나가는 힘

                    let pushForce = hammerCol.overlap * PUSH_POWER;
                    let swingForce = -velAlongNormal * SWING_POWER; 
                    
                    let totalNormalForce = pushForce + Math.max(0, swingForce);
                    
                    player.vx += hammerCol.nx * totalNormalForce * 1.2;
                    player.vy += hammerCol.ny * totalNormalForce;

                    // 지렛대 원리 / 접지력 (Traction Force)
                    let tx = -hammerCol.ny;
                    let ty = hammerCol.nx;
                    
                    let vTan = hammerVelX * tx + hammerVelY * ty;
                    
                    let mat = materials[hammerCol.type] || materials[0];
                    let friction = hammer.friction * mat.friction;
                    
                    const LEVER_POWER = 0.45; 
                    let leverForce = -vTan * friction * LEVER_POWER;
                    
                    player.vx += tx * leverForce;
                    player.vy += ty * leverForce;
                }

                let dx = player.x - hammer.x;
                let dy = player.y - hammer.y;
                let dist = Math.sqrt(dx*dx + dy*dy);
                if (dist < hammer.length - 2) {
                     let fix = (hammer.length - dist) * 0.5;
                     let fixX = (dx / dist) * fix;
                     let fixY = (dy / dist) * fix;
                     player.x += fixX;
                     player.y += fixY;
                     
                     player.vx += fixX * 0.2; 
                     player.vy += fixY * 0.2;
                }

            } else {
                hammer.x = idealHx;
                hammer.y = idealHy;
            }

            player.x += player.vx;
            player.y += player.vy;

            // 플레이어 몸체 충돌
            let bodyCol = checkCollision(player.x, player.y, player.radius);
            if (bodyCol.hit) {
                player.x += bodyCol.nx * bodyCol.overlap;
                player.y += bodyCol.ny * bodyCol.overlap;

                let vDotN = player.vx * bodyCol.nx + player.vy * bodyCol.ny;
                if (vDotN < 0) {
                    let mat = materials[bodyCol.type] || materials[0];
                    let j = -(1 + mat.bounce) * vDotN;
                    player.vx += j * bodyCol.nx;
                    player.vy += j * bodyCol.ny;

                    let tx = -bodyCol.ny;
                    let ty = bodyCol.nx;
                    let vDotT = player.vx * tx + player.vy * ty;
                    
                    player.vx -= vDotT * tx * mat.friction * 0.1;
                    player.vy -= vDotT * ty * mat.friction * 0.1;
                }
            }

            // 카메라 업데이트
            let targetCamX = player.x - width * 0.3;
            if (targetCamX < 0) targetCamX = 0;
            let targetCamY = player.y - height * 0.6;
            
            cameraX += (targetCamX - cameraX) * 0.1;
            cameraY += (targetCamY - cameraY) * 0.1;

            updateLocationText(player.x);
        }
        
        function updateLocationText(x) {
            let text = "";
            if (x < 1000) text = "출발지 (충주 입성)";
            else if (x < 1800) text = "충주 사과 농장 🍎";
            else if (x < 2800) text = "탄금대 징검다리 🌉";
            else if (x < 3800) text = "충주호 절벽 🏞️";
            else text = "중앙탑 (최종 목적지) 🗿";
            locDisplay.innerText = "현재 위치: " + text;
        }

        function drawBackground() {
            let grad = ctx.createLinearGradient(0, 0, 0, height);
            grad.addColorStop(0, "#87CEEB"); grad.addColorStop(1, "#E0F7FA"); 
            ctx.fillStyle = grad;
            ctx.fillRect(0, 0, width, height);

            ctx.save();
            ctx.translate(-cameraX * 0.3, -cameraY * 0.2); 
            ctx.fillStyle = "rgba(60, 179, 113, 0.5)";
            ctx.beginPath(); 
            ctx.moveTo(-100, height + 500); 
            ctx.lineTo(500, height - 300); 
            ctx.lineTo(1500, height + 500); 
            ctx.lineTo(3000, height - 500);
            ctx.lineTo(5000, height + 500);
            ctx.fill();
            ctx.fillStyle = "rgba(255, 255, 255, 0.4)";
            ctx.beginPath(); ctx.arc(800, 200, 60, 0, Math.PI*2); ctx.fill();
            ctx.beginPath(); ctx.arc(2000, 150, 80, 0, Math.PI*2); ctx.fill();
            ctx.restore();
        }

        function draw() {
            drawBackground();
            ctx.save();
            ctx.translate(-cameraX, -cameraY);

            for (let p of platforms) {
                if (p.type === 0) ctx.fillStyle = '#6d4c41'; 
                else if (p.type === 1) ctx.fillStyle = '#90a4ae'; 
                else if (p.type === 2) ctx.fillStyle = '#8d6e63'; 
                
                ctx.fillRect(p.x, p.y, p.w, p.h);
                ctx.strokeStyle = "rgba(0,0,0,0.3)"; ctx.lineWidth = 2; ctx.strokeRect(p.x, p.y, p.w, p.h);
                if (p.type === 0 && p.w > 50) { ctx.fillStyle = '#4caf50'; ctx.fillRect(p.x, p.y, p.w, 15); }
                
                if (p.type === 2 && p.w > 100) {
                     drawApple(p.x + 50, p.y - 10);
                     drawApple(p.x + p.w - 50, p.y - 10);
                }
            }

            ctx.save();
            ctx.translate(player.x, player.y);
            if (imageLoaded) {
                const size = player.radius * 5; 
                ctx.rotate(player.vx * 0.05);
                ctx.shadowColor = "rgba(0,0,0,0.4)"; ctx.shadowBlur = 15; ctx.shadowOffsetY = 15;
                ctx.drawImage(charImage, -size/2, -size/2, size, size);
                ctx.shadowBlur = 0;
            } else {
                ctx.rotate(player.vx * 0.05);
                const s = 2.5; 
                ctx.fillStyle = "#333"; ctx.beginPath(); ctx.ellipse(0, 10, 30*s, 40*s, 0, 0, Math.PI*2); ctx.fill();
                ctx.fillStyle = "white"; ctx.beginPath(); ctx.moveTo(-10*s, -20*s); ctx.lineTo(10*s, -20*s); ctx.lineTo(0, 10*s); ctx.fill();
                ctx.fillStyle = "#ffccbc"; ctx.beginPath(); ctx.arc(0, -40*s, 25*s, 0, Math.PI*2); ctx.fill();
                ctx.strokeStyle = "black"; ctx.lineWidth = 3; ctx.strokeRect(-20, -110, 15, 10); ctx.strokeRect(5, -110, 15, 10); ctx.moveTo( -5, -105); ctx.lineTo(5, -105); ctx.stroke();
            }
            ctx.restore();

            ctx.beginPath(); ctx.moveTo(player.x, player.y); ctx.lineTo(hammer.x, hammer.y);
            ctx.strokeStyle = '#5d4037'; ctx.lineWidth = 10; ctx.lineCap = 'round'; ctx.stroke();

            ctx.save();
            ctx.translate(hammer.x, hammer.y);
            ctx.rotate(player.angle);
            ctx.fillStyle = '#333'; ctx.fillRect(-15, -40, 30, 80);
            ctx.strokeStyle = '#111'; ctx.lineWidth = 3; ctx.strokeRect(-15, -40, 30, 80);
            ctx.fillStyle = 'rgba(255,255,255,0.3)'; ctx.fillRect(-10, -30, 8, 60);
            ctx.restore();
            ctx.restore();
            
            // 조준점 (Aim Guide) 그리기 (전체 화면 편의성)
            // 카메라 변환을 다시 되돌린 후 UI 레이어처럼 그릴 수도 있지만,
            // 게임 월드 내에서 플레이어와 마우스 사이의 선을 긋는 것이 더 직관적임.
            ctx.restore(); // 카메라 변환 종료 (화면 좌표계로 복귀)

            // 조준 가이드라인 (플레이어 -> 마우스)
            let screenPx = player.x - cameraX;
            let screenPy = player.y - cameraY;
            
            ctx.beginPath();
            ctx.moveTo(screenPx, screenPy);
            ctx.lineTo(mouseX, mouseY);
            ctx.strokeStyle = "rgba(255, 0, 0, 0.3)";
            ctx.lineWidth = 2;
            ctx.setLineDash([5, 5]);
            ctx.stroke();
            ctx.setLineDash([]);
            
            // 조준점 (마우스 위치)
            ctx.beginPath();
            ctx.arc(mouseX, mouseY, 10, 0, Math.PI * 2);
            ctx.strokeStyle = "red";
            ctx.lineWidth = 2;
            ctx.stroke();
            ctx.fillStyle = "rgba(255, 0, 0, 0.2)";
            ctx.fill();

            requestAnimationFrame(loop);
        }

        function drawApple(x, y) {
            ctx.save();
            ctx.translate(x, y);
            ctx.fillStyle = "#FF4444"; ctx.beginPath(); ctx.arc(0, 0, 15, 0, Math.PI*2); ctx.fill();
            ctx.fillStyle = "#4CAF50"; ctx.beginPath(); ctx.ellipse(5, -10, 6, 3, Math.PI/4, 0, Math.PI*2); ctx.fill();
            ctx.restore();
        }

        function loop() { update(); draw(); }
        resize(); resetGame(); loop();
    </script>
</body>
</html>
