<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>충주맨의 항아리 등반 (충주시청 에디션)</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #87CEEB;
            font-family: 'Noto Sans KR', sans-serif;
            user-select: none;
            touch-action: none; 
            cursor: crosshair;
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
            pointer-events: auto;
            z-index: 10;
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
            z-index: 5;
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
            z-index: 5;
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
        
        #ending-layer {
            display: none;
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: 200;
            background: rgba(0,0,0,0.3);
            flex-direction: column;
            justify-content: center;
            align-items: center;
        }
        #ending-title {
            position: absolute;
            top: 15%;
            width: 100%;
            text-align: center;
            font-size: 3rem;
            font-weight: 900;
            color: white;
            text-shadow: 2px 2px 10px rgba(0,0,0,0.8);
            z-index: 202;
            pointer-events: none;
        }
        .choice-container {
            display: flex;
            width: 100%;
            height: 100%;
        }
        .choice-btn {
            flex: 1;
            display: flex;
            justify-content: center;
            align-items: center;
            font-size: 2rem;
            font-weight: bold;
            color: white;
            cursor: pointer;
            transition: all 0.5s ease;
            opacity: 0.8;
        }
        .choice-btn:hover {
            opacity: 1;
            flex: 1.2;
        }
        .choice-btn.red {
            background-color: rgba(255, 0, 0, 0.6);
        }
        .choice-btn.blue {
            background-color: rgba(0, 0, 255, 0.6);
        }
        .choice-btn.expanded {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: 300;
            opacity: 1;
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

    <!-- 엔딩 레이어 -->
    <div id="ending-layer">
        <div id="ending-title">다음은 충주시장?</div>
        <div class="choice-container">
            <div id="btn-red" class="choice-btn red" onclick="selectEnding('red')"></div>
            <div id="btn-blue" class="choice-btn blue" onclick="selectEnding('blue')"></div>
        </div>
    </div>

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
        const endingLayer = document.getElementById('ending-layer');
        const btnRed = document.getElementById('btn-red');
        const btnBlue = document.getElementById('btn-blue');

        // === 게임 설정 ===
        let width, height;
        const gravity = 0.8; 
        const airDrag = 0.99; 
        
        let cameraX = 0;
        let cameraY = 0;
        let isGameOver = false;
        let isGameClear = false;

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
            0: { friction: 0.7, bounce: 0.1, name: "바위/콘크리트" },
            1: { friction: 0.1, bounce: 0.05, name: "철/얼음" },
            2: { friction: 0.9, bounce: 0.1, name: "나무/상자" } // 잡기 매우 쉬움
        };

        // === [맵 레벨 디자인: 충주 명물 & 시청] ===
        // type 2(나무/상자)는 마찰력이 높아 잡기 쉬움
        // type 0(바위/콘크리트)는 보통
        // 간격을 촘촘하게 하여 난이도 하락, but 떨어지면 끝
        const platforms = [
            // [구역 0: 시작 지점 (공중 부양)]
            { x: 200, y: -40, w: 80, h: 40, type: 2 }, // 착지점
            { x: 300, y: -80, w: 80, h: 40, type: 2 },
            { x: 400, y: -120, w: 60, h: 30, type: 2 },
            { x: 500, y: -160, w: 80, h: 40, type: 2 },
            { x: 600, y: -200, w: 60, h: 30, type: 2 },
            { x: 750, y: -150, w: 100, h: 30, type: 0 }, // 돌 턱

            // [구역 1: 충주 사과 과수원 (잔가지 지옥 - 잡기 쉬움)]
            { x: 900, y: -100, w: 150, h: 40, type: 2 }, // 과수원 입구
            
            // 촘촘한 사과 상자와 가지들
            { x: 1100, y: -50, w: 60, h: 20, type: 2 },
            { x: 1150, y: -100, w: 60, h: 20, type: 2 },
            { x: 1200, y: -150, w: 60, h: 20, type: 2 },
            { x: 1250, y: -200, w: 60, h: 20, type: 2 },
            { x: 1300, y: -250, w: 60, h: 20, type: 2 },
            
            { x: 1400, y: -300, w: 200, h: 50, type: 2 }, // 큰 나무
            { x: 1450, y: -380, w: 50, h: 20, type: 2 }, // 윗 가지
            { x: 1550, y: -420, w: 50, h: 20, type: 2 }, 
            
            { x: 1700, y: -350, w: 150, h: 40, type: 2 }, // 연결 다리
            { x: 1800, y: -450, w: 50, h: 20, type: 2 }, // 뜬금 없는 상자

            // [구역 2: 탄금대 바위 지대 (약간 미끄러움)]
            { x: 1950, y: -400, w: 100, h: 50, type: 0 },
            { x: 2050, y: -450, w: 60, h: 40, type: 0 },
            { x: 2120, y: -500, w: 60, h: 40, type: 0 },
            { x: 2200, y: -550, w: 80, h: 30, type: 0 },
            { x: 2300, y: -600, w: 200, h: 50, type: 0 }, // 중간 휴식처
            { x: 2400, y: -700, w: 50, h: 50, type: 0 }, // 뾰족 바위
            { x: 2500, y: -650, w: 100, h: 30, type: 0 },

            // [구역 3: 충주시청 (콘크리트 빌딩 숲)]
            // 계단식 구조로 배치된 콘크리트 슬래브
            { x: 2700, y: -700, w: 600, h: 300, type: 0 }, // 시청 본관 하부 (거대한 벽)
            
            // 시청 외벽 등반 (창틀 느낌의 홀드)
            { x: 2720, y: -750, w: 60, h: 20, type: 0 }, 
            { x: 2800, y: -800, w: 60, h: 20, type: 0 },
            { x: 2750, y: -850, w: 60, h: 20, type: 0 },
            { x: 2850, y: -900, w: 60, h: 20, type: 0 },
            { x: 2950, y: -950, w: 60, h: 20, type: 0 },
            
            { x: 3100, y: -1000, w: 300, h: 40, type: 0 }, // 시청 옥상 테라스
            { x: 3200, y: -1100, w: 100, h: 20, type: 0 }, // 옥상 구조물
            { x: 3300, y: -1150, w: 50, h: 50, type: 2 },  // 옥상 위 사과 상자(서류 상자?)
            
            // 시청에서 중앙탑으로 가는 구름다리 (매우 위험)
            { x: 3500, y: -1100, w: 50, h: 20, type: 1 }, // 철근 1 (미끄러움)
            { x: 3600, y: -1150, w: 50, h: 20, type: 1 }, // 철근 2
            { x: 3700, y: -1100, w: 50, h: 20, type: 1 }, // 철근 3

            // [구역 4: 중앙탑 (최종 등반)]
            { x: 3900, y: -1200, w: 800, h: 100, type: 0 }, // 탑 광장
            
            // 탑 등반 (촘촘한 기단)
            { x: 4100, y: -1250, w: 80, h: 30, type: 0 },
            { x: 4150, y: -1300, w: 80, h: 30, type: 0 },
            { x: 4200, y: -1350, w: 80, h: 30, type: 0 },
            { x: 4250, y: -1400, w: 80, h: 30, type: 0 },
            
            { x: 4400, y: -1450, w: 300, h: 40, type: 0 }, // 기단 상부
            { x: 4420, y: -1520, w: 40, h: 20, type: 0 },  // 홀드
            { x: 4480, y: -1580, w: 40, h: 20, type: 0 },  // 홀드
            
            { x: 4450, y: -1650, w: 150, h: 150, type: 0 }, // 탑신 (도착점)
            { x: 4500, y: -1800, w: 50, h: 150, type: 0 }, // 상륜부
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
        
        function updateInput(x, y) {
            mouseX = x;
            mouseY = y;
        }

        window.addEventListener('mousemove', e => {
            updateInput(e.clientX, e.clientY);
        });

        window.addEventListener('touchstart', e => {
            if(e.touches.length > 0) {
                updateInput(e.touches[0].clientX, e.touches[0].clientY);
            }
        }, {passive: false});

        window.addEventListener('touchmove', e => {
            if(e.touches.length > 0) {
                e.preventDefault(); 
                updateInput(e.touches[0].clientX, e.touches[0].clientY);
            }
        }, {passive: false});


        function resetGame() {
            isGameOver = false;
            isGameClear = false;
            gameOverMsg.style.display = 'none';
            endingLayer.style.display = 'none';
            btnRed.classList.remove('expanded');
            btnBlue.classList.remove('expanded');
            btnRed.style.display = 'flex';
            btnBlue.style.display = 'flex';

            player.x = 220;
            player.y = -300; 
            player.vx = 0;
            player.vy = 0;
            cameraX = 0;
            cameraY = 0;
            player.prevAngle = 0;
            
            mouseX = width / 2;
            mouseY = height / 2;
        }

        function gameOver() {
            if(isGameOver || isGameClear) return;
            isGameOver = true;
            gameOverMsg.style.display = 'block';
            setTimeout(() => { resetGame(); }, 2500);
        }

        function gameClear() {
            if(isGameClear || isGameOver) return;
            isGameClear = true;
            endingLayer.style.display = 'flex';
        }

        function selectEnding(color) {
            if (color === 'red') {
                btnRed.classList.add('expanded');
                btnBlue.style.display = 'none';
            } else {
                btnBlue.classList.add('expanded');
                btnRed.style.display = 'none';
            }
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

        let particles = [];
        function createFirework() {
            if (Math.random() < 0.1) {
                let px = Math.random() * width;
                let py = Math.random() * height;
                let color = `hsl(${Math.random() * 360}, 100%, 50%)`;
                for (let i = 0; i < 20; i++) {
                    particles.push({
                        x: px, y: py,
                        vx: (Math.random() - 0.5) * 10,
                        vy: (Math.random() - 0.5) * 10,
                        life: 100,
                        color: color
                    });
                }
            }
        }

        function update() {
            if(isGameClear) {
                createFirework();
                for (let i = particles.length - 1; i >= 0; i--) {
                    let p = particles[i];
                    p.x += p.vx;
                    p.y += p.vy;
                    p.life--;
                    if (p.life <= 0) particles.splice(i, 1);
                }
                return;
            }

            if(isGameOver) return;

            player.vy += gravity;
            player.vx *= airDrag;
            player.vy *= airDrag;

            if (player.y > 400) gameOver();
            
            // 엔딩 조건 체크 (중앙탑 최상단)
            if (player.x > 4400 && player.y < -1650) {
                gameClear();
            }

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
            
            let hammerTanX = -Math.sin(player.angle);
            let hammerTanY = Math.cos(player.angle);
            let hammerVelX = player.vx + hammerTanX * angularVelocity * hammer.length;
            let hammerVelY = player.vy + hammerTanY * angularVelocity * hammer.length;

            let hammerCol = checkCollision(idealHx, idealHy, hammer.headRadius);

            if (hammerCol.hit) {
                hammer.x = idealHx + hammerCol.nx * hammerCol.overlap;
                hammer.y = idealHy + hammerCol.ny * hammerCol.overlap;

                let velAlongNormal = hammerVelX * hammerCol.nx + hammerVelY * hammerCol.ny;

                if (velAlongNormal < 0 || hammerCol.overlap > 0) {
                    
                    const PUSH_POWER = 0.08; 
                    const SWING_POWER = 0.15; 

                    let pushForce = hammerCol.overlap * PUSH_POWER;
                    let swingForce = -velAlongNormal * SWING_POWER; 
                    
                    let totalNormalForce = pushForce + Math.max(0, swingForce);
                    
                    player.vx += hammerCol.nx * totalNormalForce * 1.2;
                    player.vy += hammerCol.ny * totalNormalForce;

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

            let targetCamX = player.x - width * 0.3;
            if (targetCamX < 0) targetCamX = 0;
            let targetCamY = player.y - height * 0.6;
            
            cameraX += (targetCamX - cameraX) * 0.1;
            cameraY += (targetCamY - cameraY) * 0.1;

            updateLocationText(player.x);
        }
        
        function updateLocationText(x) {
            let text = "";
            if (x < 1000) text = "출발지 (사과 과수원) 🍎";
            else if (x < 1900) text = "탄금대 바위 지대 🧗";
            else if (x < 2700) text = "충주시청 (진입) 🏢";
            else if (x < 3800) text = "충주시청 (옥상) 🏢";
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
                // 플랫폼 색상 및 스타일
                if (p.type === 0) { // 콘크리트/바위
                    ctx.fillStyle = '#6d4c41'; 
                    // 시청 건물(큰 덩어리)은 회색 콘크리트 느낌
                    if (p.w > 200 && p.x > 2600 && p.x < 3500) ctx.fillStyle = '#7f8c8d';
                }
                else if (p.type === 1) ctx.fillStyle = '#90a4ae'; // 철
                else if (p.type === 2) ctx.fillStyle = '#8d6e63'; // 나무/상자
                
                ctx.fillRect(p.x, p.y, p.w, p.h);
                ctx.strokeStyle = "rgba(0,0,0,0.3)"; ctx.lineWidth = 2; ctx.strokeRect(p.x, p.y, p.w, p.h);
                
                // 장식: 사과 (과수원 구역)
                if (p.x < 1800 && p.type === 2 && p.w > 40) {
                     drawApple(p.x + p.w/2, p.y - 10);
                }
                // 장식: 시청 창문 (시청 구역 큰 벽)
                if (p.w > 300 && p.x > 2600 && p.x < 3000) {
                    ctx.fillStyle = '#bdc3c7';
                    for(let i=0; i<5; i++) {
                        for(let j=0; j<3; j++) {
                             ctx.fillRect(p.x + 50 + i*100, p.y + 50 + j*80, 40, 40);
                        }
                    }
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
            
            if(isGameClear) {
                for (let p of particles) {
                    ctx.fillStyle = p.color;
                    ctx.beginPath();
                    ctx.arc(p.x, p.y, 4, 0, Math.PI * 2);
                    ctx.fill();
                }
            }

            if(!isGameClear && !isGameOver) {
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
                
                ctx.beginPath();
                ctx.arc(mouseX, mouseY, 10, 0, Math.PI * 2);
                ctx.strokeStyle = "red";
                ctx.lineWidth = 2;
                ctx.stroke();
                ctx.fillStyle = "rgba(255, 0, 0, 0.2)";
                ctx.fill();
            }

            requestAnimationFrame(loop);
        }

        function drawApple(x, y) {
            ctx.save();
            ctx.translate(x, y);
            ctx.fillStyle = "#FF4444"; ctx.beginPath(); ctx.arc(0, 0, 12, 0, Math.PI*2); ctx.fill(); // 사과 크기 조금 줄임
            ctx.fillStyle = "#4CAF50"; ctx.beginPath(); ctx.ellipse(4, -8, 5, 2, Math.PI/4, 0, Math.PI*2); ctx.fill();
            ctx.restore();
        }

        function loop() { update(); draw(); }
        resize(); resetGame(); loop();
    </script>
</body>
</html>
