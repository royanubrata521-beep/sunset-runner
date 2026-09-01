<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Endless Runner - Beautiful Sunset</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #111;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: 'Courier New', Courier, monospace;
            overflow: hidden;
        }

        #game-container {
            position: relative;
            box-shadow: 0 0 30px rgba(0,0,0,0.9);
            border-radius: 8px;
            overflow: hidden;
        }

        canvas {
            background-color: #ffc371;
            display: block;
        }

        #ui-board {
            position: absolute;
            top: 20px;
            left: 20px;
            right: 20px;
            display: flex;
            justify-content: space-between;
            font-size: 24px;
            font-weight: bold;
            color: #fff;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
            user-select: none;
            pointer-events: none;
        }

        .overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(15, 10, 30, 0.85);
            display: none;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            color: white;
            text-align: center;
            user-select: none;
        }

        .overlay h1 {
            font-size: 64px;
            margin-bottom: 10px;
            text-transform: uppercase;
            letter-spacing: 5px;
            color: #ff7e67;
            text-shadow: 3px 3px 0 #5a1835;
        }

        .overlay p {
            font-size: 24px;
            margin-bottom: 30px;
            color: #ffe6fa;
        }

        .overlay button {
            padding: 15px 40px;
            font-size: 24px;
            font-family: inherit;
            cursor: pointer;
            border: 3px solid #ffc371;
            background-color: transparent;
            color: #ffc371;
            text-transform: uppercase;
            border-radius: 8px;
            transition: all 0.3s;
        }

        .overlay button:hover {
            background-color: #ffc371;
            color: #111;
            box-shadow: 0 0 15px #ffc371;
        }
        
        #start-screen { display: flex; }
        #start-screen h1 { color: #ffc371; text-shadow: 3px 3px 0 #d35400; }
    </style>
</head>
<body>

<div id="game-container">
    <canvas id="gameCanvas" width="1000" height="600"></canvas>
    <div id="ui-board">
        <div>🏃 DISTANCE: <span id="distance">0</span>m</div>
        <div>💰 COINS: <span id="score">0</span></div>
    </div>
    
    <!-- Start Screen -->
    <div id="start-screen" class="overlay">
        <h1>SUNSET RUNNER</h1>
        <p>Press SPACE or UP ARROW to Jump</p>
        <button onclick="startGame()">Start Game</button>
    </div>

    <!-- Game Over Screen -->
    <div id="game-over-screen" class="overlay">
        <h1>GAME OVER</h1>
        <p>You ran <span id="final-distance">0</span>m and collected <span id="final-score">0</span> coins!</p>
        <button onclick="startGame()">Run Again</button>
    </div>
</div>

<script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    const scoreElement = document.getElementById("score");
    const distanceElement = document.getElementById("distance");
    const gameOverScreen = document.getElementById("game-over-screen");
    const startScreen = document.getElementById("start-screen");

    // Game State
    let animationId;
    let gameState = "start"; 
    let frameCount = 0;
    
    // Core variables (Kept SLOW as requested)
    let gameSpeed = 4; 
    let distance = 0;
    let coinsCollected = 0;
    let spawnTimer = 0;

    // Physics
    const gravity = 0.7;
    
    // World
    const ground = {
        y: 530,
        height: 70
    };

    // Entities Arrays
    let player = {};
    let monsters = [];
    let platforms = [];
    let coins = [];
    let scenery = { mountains: [], clouds: [], trees: [], fireflies: [], groundStripes: [] };

    // Jump Input
    window.addEventListener("keydown", (e) => {
        if ((e.code === "Space" || e.code === "ArrowUp") && gameState === "playing") {
            e.preventDefault();
            if (player.grounded) {
                player.dy = player.jumpPower;
                player.grounded = false;
            }
        }
    });

    window.addEventListener("touchstart", (e) => {
        if (gameState === "playing" && player.grounded) {
            player.dy = player.jumpPower;
            player.grounded = false;
        }
    });

    function initGame() {
        gameSpeed = 4; // Slow mode
        distance = 0;
        coinsCollected = 0;
        frameCount = 0;
        spawnTimer = 60; 
        
        scoreElement.innerText = coinsCollected;
        distanceElement.innerText = Math.floor(distance);

        player = {
            x: 150,
            y: ground.y - 60,
            width: 32,
            height: 55,
            colorShirt: "#e74c3c",
            colorPants: "#3498db",
            colorSkin: "#fcd5b5",
            colorHair: "#6f4e37",
            dy: 0,
            jumpPower: -15,
            grounded: true
        };

        monsters = [];
        platforms = [];
        coins = [];
        
        // Beautiful Scenery Setup
        scenery.mountains = [
            // Background (lighter, blends with sky)
            { x: 0, width: 450, height: 280, color: "#874b78" },
            { x: 400, width: 550, height: 350, color: "#723d6a" },
            { x: 900, width: 400, height: 260, color: "#874b78" },
            // Foreground (darker)
            { x: 200, width: 400, height: 200, color: "#4a2c5a" },
            { x: 700, width: 500, height: 220, color: "#3d224d" }
        ];

        scenery.clouds = [
            { x: 100, y: 120, scale: 1.2 }, { x: 450, y: 80, scale: 1.5 },
            { x: 850, y: 140, scale: 1.0 }, { x: 1200, y: 90, scale: 1.3 }
        ];

        scenery.trees = [
            { x: 200, scale: 1.0 }, { x: 450, scale: 0.85 }, 
            { x: 750, scale: 1.2 }, { x: 1050, scale: 0.9 },
            { x: 1350, scale: 1.1 }
        ];

        scenery.fireflies = [];
        for (let i = 0; i < 20; i++) {
            scenery.fireflies.push({
                x: Math.random() * canvas.width,
                y: Math.random() * ground.y,
                size: Math.random() * 2 + 1,
                speed: Math.random() * 1 + 0.2,
                offset: Math.random() * 100
            });
        }

        scenery.groundStripes = [];
        for (let i = 0; i < 20; i++) scenery.groundStripes.push(i * 60);
    }

    function spawnObstacles() {
        spawnTimer--;
        if (spawnTimer <= 0) {
            let rand = Math.random();
            let spawnX = canvas.width + 50;

            if (rand < 0.35) {
                spawnMonster(spawnX, ground.y - 70);
                spawnTimer = Math.random() * 80 + 80; 
            } 
            else if (rand < 0.70) {
                let platY = ground.y - (Math.random() * 100 + 100); 
                let platWidth = Math.random() * 100 + 100; 
                platforms.push({ x: spawnX, y: platY, width: platWidth, height: 20 });
                
                if (Math.random() > 0.5) {
                    spawnCoins(spawnX + 20, platY - 25, Math.floor(platWidth / 40));
                } else {
                    spawnMonster(spawnX + platWidth / 2 - 30, platY - 70);
                }
                spawnTimer = Math.random() * 60 + 80; 
            } 
            else {
                spawnCoins(spawnX, ground.y - 25, Math.floor(Math.random() * 3 + 2));
                spawnTimer = Math.random() * 50 + 50;
            }
        }
    }

    function spawnMonster(x, y) {
        monsters.push({
            x: x,
            y: y,
            width: 60,
            height: 70,
            color: "#1a1a2e", // Darker monster to fit sunset lighting
            colorSpikes: "#e94560", 
            speedOffset: Math.random() * 1.5 + 0.5 
        });
    }

    function spawnCoins(startX, y, count) {
        for (let i = 0; i < count; i++) {
            coins.push({ x: startX + (i * 40), y: y, radius: 12 });
        }
    }

    function update() {
        frameCount++;
        
        gameSpeed += 0.001; 
        distance += gameSpeed / 60;
        distanceElement.innerText = Math.floor(distance);

        player.dy += gravity;
        player.y += player.dy;

        player.grounded = false;
        if (player.y + player.height >= ground.y) {
            player.y = ground.y - player.height;
            player.dy = 0;
            player.grounded = true;
        }

        platforms.forEach(plat => {
            if (player.dy > 0 && player.x + player.width > plat.x && player.x < plat.x + plat.width) {
                let playerBottom = player.y + player.height;
                let prevPlayerBottom = playerBottom - player.dy;
                if (prevPlayerBottom <= plat.y && playerBottom >= plat.y) {
                    player.y = plat.y - player.height;
                    player.dy = 0;
                    player.grounded = true;
                }
            }
        });

        spawnObstacles();

        // Scroll Entities
        platforms.forEach(p => p.x -= gameSpeed);
        coins.forEach(c => c.x -= gameSpeed);
        monsters.forEach(m => m.x -= (gameSpeed + m.speedOffset));
        
        // Scroll Parallax Scenery
        scenery.mountains.forEach(m => { m.x -= gameSpeed * 0.15; if (m.x + m.width < 0) m.x = canvas.width + Math.random() * 100; });
        scenery.clouds.forEach(c => { c.x -= gameSpeed * 0.3; if (c.x + 100 < 0) c.x = canvas.width + Math.random() * 200; });
        scenery.trees.forEach(t => { t.x -= gameSpeed * 0.8; if (t.x + 100 < 0) t.x = canvas.width + Math.random() * 200; });
        
        scenery.fireflies.forEach(f => {
            f.x -= (gameSpeed * 0.5 + f.speed);
            f.y += Math.sin(frameCount * 0.05 + f.offset) * 0.5;
            if (f.x < -10) { f.x = canvas.width + 10; f.y = Math.random() * ground.y; }
        });

        for(let i=0; i<scenery.groundStripes.length; i++) {
            scenery.groundStripes[i] -= gameSpeed;
            if (scenery.groundStripes[i] < -50) scenery.groundStripes[i] += canvas.width + 50;
        }

        platforms = platforms.filter(p => p.x + p.width > -100);
        coins = coins.filter(c => c.x + c.radius > -50);
        monsters = monsters.filter(m => m.x + m.width > -100);

        for (let i = coins.length - 1; i >= 0; i--) {
            let c = coins[i];
            let coinBox = { x: c.x - c.radius, y: c.y - c.radius, width: c.radius * 2, height: c.radius * 2 };
            if (rectIntersect(player, coinBox)) {
                coinsCollected++;
                scoreElement.innerText = coinsCollected;
                coins.splice(i, 1);
            }
        }

        for (let m of monsters) {
            let hitBox = { x: m.x + 10, y: m.y - 10, width: m.width - 20, height: m.height + 10 };
            if (rectIntersect(player, hitBox)) {
                endGame();
                return;
            }
        }
    }

    function rectIntersect(r1, r2) {
        return !(r2.x > r1.x + r1.width || r2.x + r2.width < r1.x || r2.y > r1.y + r1.height || r2.y + r2.height < r1.y);
    }

    // --- Drawing Functions ---
    function draw() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        // 1. Beautiful Sunset Sky Gradient
        let skyGradient = ctx.createLinearGradient(0, 0, 0, ground.y);
        skyGradient.addColorStop(0, "#2b1055"); // Deep purple top
        skyGradient.addColorStop(0.4, "#7597de"); // Purple/blue mid
        skyGradient.addColorStop(0.7, "#ff7e5f"); // Vivid sunset orange
        skyGradient.addColorStop(1, "#feb47b"); // Yellowish horizon
        ctx.fillStyle = skyGradient;
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        // 2. Glowing Sun
        drawSun();

        // 3. Background Layers
        drawMountains();
        drawClouds();
        drawTrees();
        drawFireflies();

        // 4. Detailed Ground
        let groundGradient = ctx.createLinearGradient(0, ground.y, 0, canvas.height);
        groundGradient.addColorStop(0, "#27ae60"); // Bright grass
        groundGradient.addColorStop(0.2, "#2ecc71"); // Lighter grass
        groundGradient.addColorStop(0.3, "#795548"); // Warm dirt
        groundGradient.addColorStop(1, "#4e342e"); // Dark deep dirt
        ctx.fillStyle = groundGradient;
        ctx.fillRect(0, ground.y, canvas.width, ground.height);
        
        ctx.fillStyle = "rgba(0,0,0,0.15)"; // Soft shadows for speed stripes
        scenery.groundStripes.forEach(x => {
            ctx.fillRect(x, ground.y + 25, 12, 6);
            ctx.fillRect(x + 20, ground.y + 40, 20, 6);
        });

        // 5. Platforms (Wood style)
        platforms.forEach(plat => {
            ctx.fillStyle = "#5d4037"; // Wood brown
            ctx.fillRect(plat.x, plat.y, plat.width, plat.height);
            ctx.strokeStyle = "#3e2723";
            ctx.lineWidth = 3;
            ctx.strokeRect(plat.x, plat.y, plat.width, plat.height);
            // Grass moss on top
            ctx.fillStyle = "#2ecc71";
            ctx.fillRect(plat.x, plat.y, plat.width, 6);
            // Highlight
            ctx.fillStyle = "rgba(255,255,255,0.2)";
            ctx.fillRect(plat.x, plat.y+6, plat.width, 2);
        });

        // 6. Coins (Glowing)
        coins.forEach(coin => {
            ctx.shadowBlur = 10;
            ctx.shadowColor = "#f1c40f";
            ctx.fillStyle = "#f1c40f";
            ctx.beginPath();
            ctx.arc(coin.x, coin.y, coin.radius, 0, Math.PI * 2);
            ctx.fill();
            
            ctx.shadowBlur = 0; // Reset
            ctx.fillStyle = "white"; // Shine
            ctx.beginPath();
            ctx.arc(coin.x - 4, coin.y - 4, 3, 0, Math.PI * 2);
            ctx.fill();
        });

        monsters.forEach(m => drawMonster(m));
        drawPlayer();
    }

    function drawSun() {
        ctx.save();
        ctx.fillStyle = "rgba(255, 230, 150, 0.9)";
        ctx.shadowBlur = 50;
        ctx.shadowColor = "#ffeb3b";
        ctx.beginPath();
        ctx.arc(canvas.width * 0.7, ground.y - 100, 60, 0, Math.PI * 2);
        ctx.fill();
        ctx.restore();
    }

    function drawMountains() {
        scenery.mountains.forEach(mtn => {
            // Mountain Base
            ctx.fillStyle = mtn.color;
            ctx.beginPath();
            ctx.moveTo(mtn.x, ground.y);
            ctx.lineTo(mtn.x + mtn.width / 2, ground.y - mtn.height);
            ctx.lineTo(mtn.x + mtn.width, ground.y);
            ctx.fill();

            // Snow cap
            let capH = mtn.height * 0.25;
            let capW = mtn.width * 0.25;
            ctx.fillStyle = "rgba(255, 240, 245, 0.8)"; // Pinkish snow
            ctx.beginPath();
            ctx.moveTo(mtn.x + mtn.width/2 - capW/2, ground.y - mtn.height + capH);
            ctx.lineTo(mtn.x + mtn.width/2, ground.y - mtn.height);
            ctx.lineTo(mtn.x + mtn.width/2 + capW/2, ground.y - mtn.height + capH);
            // Jagged snow bottom
            ctx.lineTo(mtn.x + mtn.width/2 + capW/4, ground.y - mtn.height + capH + 15);
            ctx.lineTo(mtn.x + mtn.width/2, ground.y - mtn.height + capH + 5);
            ctx.lineTo(mtn.x + mtn.width/2 - capW/4, ground.y - mtn.height + capH + 15);
            ctx.fill();
        });
    }

    function drawClouds() {
        ctx.fillStyle = "rgba(255, 230, 250, 0.6)"; // Translucent pinkish clouds
        scenery.clouds.forEach(c => {
            ctx.beginPath();
            ctx.arc(c.x, c.y, 25*c.scale, 0, Math.PI*2);
            ctx.arc(c.x+30*c.scale, c.y-15*c.scale, 35*c.scale, 0, Math.PI*2);
            ctx.arc(c.x+60*c.scale, c.y, 25*c.scale, 0, Math.PI*2);
            ctx.arc(c.x+30*c.scale, c.y+10*c.scale, 20*c.scale, 0, Math.PI*2);
            ctx.fill();
        });
    }

    function drawTrees() {
        scenery.trees.forEach(t => {
            let s = t.scale;
            // Trunk
            ctx.fillStyle = "#3e2723"; 
            ctx.fillRect(t.x - 6*s, ground.y - 30*s, 12*s, 30*s);
            
            // Layered Pine Leaves (3 layers)
            ctx.fillStyle = "#1b5e20"; // Dark rich green
            
            // Bottom layer
            ctx.beginPath(); ctx.moveTo(t.x, ground.y - 70*s);
            ctx.lineTo(t.x - 35*s, ground.y - 20*s); ctx.lineTo(t.x + 35*s, ground.y - 20*s); ctx.fill();
            // Middle layer
            ctx.beginPath(); ctx.moveTo(t.x, ground.y - 95*s);
            ctx.lineTo(t.x - 30*s, ground.y - 45*s); ctx.lineTo(t.x + 30*s, ground.y - 45*s); ctx.fill();
            // Top layer
            ctx.beginPath(); ctx.moveTo(t.x, ground.y - 120*s);
            ctx.lineTo(t.x - 25*s, ground.y - 70*s); ctx.lineTo(t.x + 25*s, ground.y - 70*s); ctx.fill();
        });
    }

    function drawFireflies() {
        ctx.fillStyle = "#e0f7fa";
        ctx.shadowBlur = 8;
        ctx.shadowColor = "#00bcd4";
        scenery.fireflies.forEach(f => {
            ctx.beginPath();
            ctx.arc(f.x, f.y, f.size, 0, Math.PI * 2);
            ctx.fill();
        });
        ctx.shadowBlur = 0; // Reset
    }

    function drawMonster(m) {
        let runBounce = Math.sin(frameCount * 0.25) * 3;
        let my = m.y + runBounce;

        // Spikes
        ctx.fillStyle = m.colorSpikes;
        ctx.beginPath();
        ctx.moveTo(m.x, my);
        ctx.lineTo(m.x + m.width/4, my - 20);
        ctx.lineTo(m.x + m.width/2, my);
        ctx.lineTo(m.x + 3*m.width/4, my - 20);
        ctx.lineTo(m.x + m.width, my);
        ctx.fill();

        // Shadow under monster
        ctx.fillStyle = "rgba(0,0,0,0.3)";
        ctx.beginPath();
        ctx.ellipse(m.x + m.width/2, m.y + m.height, m.width/2, 5, 0, 0, Math.PI*2);
        ctx.fill();

        // Body
        ctx.fillStyle = m.color;
        ctx.fillRect(m.x, my, m.width, m.height);
        
        // Eyes
        ctx.fillStyle = "#ff4757"; // Glowing red
        ctx.beginPath(); ctx.arc(m.x + 15, my + 20, 8, 0, Math.PI*2); ctx.fill();
        ctx.beginPath(); ctx.arc(m.x + 35, my + 20, 8, 0, Math.PI*2); ctx.fill();
        ctx.fillStyle = "#111";
        ctx.beginPath(); ctx.arc(m.x + 13, my + 20, 3, 0, Math.PI*2); ctx.fill(); 
        ctx.beginPath(); ctx.arc(m.x + 33, my + 20, 3, 0, Math.PI*2); ctx.fill();

        // Mouth
        ctx.fillStyle = "#fff";
        ctx.beginPath();
        ctx.moveTo(m.x+10, my+40); ctx.lineTo(m.x+15, my+35); ctx.lineTo(m.x+20, my+40);
        ctx.lineTo(m.x+25, my+35); ctx.lineTo(m.x+30, my+40); ctx.lineTo(m.x+35, my+35);
        ctx.lineTo(m.x+40, my+40); ctx.lineTo(m.x+35, my+45); ctx.lineTo(m.x+30, my+40);
        ctx.lineTo(m.x+25, my+45); ctx.lineTo(m.x+20, my+40); ctx.lineTo(m.x+15, my+45);
        ctx.fill();
    }

    function drawPlayer() {
        let px = player.x;
        let py = player.y;
        let pw = player.width;
        let ph = player.height;

        let legSwing = player.grounded ? Math.sin(frameCount * 0.2) * 12 : 0;
        let armSwing = player.grounded ? Math.sin(frameCount * 0.2) * 10 : -15;
        let bodyBounce = player.grounded ? Math.abs(Math.sin(frameCount * 0.2)) * 3 : 0;
        
        py += bodyBounce; 

        // Player Shadow
        if (player.grounded) {
            ctx.fillStyle = "rgba(0,0,0,0.3)";
            ctx.beginPath();
            ctx.ellipse(px + pw/2, ground.y, pw/1.5, 4, 0, 0, Math.PI*2);
            ctx.fill();
        }

        // Back Arm
        ctx.fillStyle = "#d3a88a"; // Slightly darker for back shadow
        ctx.fillRect(px + pw/2 - 4 + armSwing, py + 15, 8, 20);

        // Back Leg
        ctx.fillStyle = "#2980b9"; // Darker blue
        ctx.fillRect(px + pw/2 - 4 - legSwing, py + ph - 25, 10, 25);

        // Body/Shirt
        ctx.fillStyle = player.colorShirt;
        ctx.fillRect(px, py + ph - 45, pw, 25);
        // Shirt shading
        ctx.fillStyle = "rgba(0,0,0,0.15)";
        ctx.fillRect(px, py + ph - 25, pw, 5);

        // Front Leg
        ctx.fillStyle = player.colorPants;
        ctx.fillRect(px + pw/2 - 6 + legSwing, py + ph - 25, 12, 25);

        // Front Arm
        ctx.fillStyle = player.colorSkin;
        ctx.fillRect(px + pw/2 - 4 - armSwing, py + 15, 8, 20);

        // Head & Hair
        ctx.fillStyle = player.colorSkin;
        ctx.fillRect(px + 4, py, pw - 4, 18); 
        ctx.fillStyle = player.colorHair;
        ctx.fillRect(px + 2, py - 4, pw - 2, 8); 
        ctx.fillRect(px + 4, py + 2, 6, 12); 

        // Eyes
        ctx.fillStyle = "white";
        ctx.fillRect(px + pw - 8, py + 4, 6, 6);
        ctx.fillStyle = "black";
        ctx.fillRect(px + pw - 5, py + 6, 3, 3);
    }

    function gameLoop() {
        if (gameState === "playing") {
            update();
            draw();
            animationId = requestAnimationFrame(gameLoop);
        }
    }

    function startGame() {
        startScreen.style.display = "none";
        gameOverScreen.style.display = "none";
        initGame();
        gameState = "playing";
        gameLoop();
    }

    function endGame() {
        gameState = "gameover";
        cancelAnimationFrame(animationId);
        
        document.getElementById("final-distance").innerText = Math.floor(distance);
        document.getElementById("final-score").innerText = coinsCollected;
        gameOverScreen.style.display = "flex";
    }

    // Initial render behind start screen
    initGame();
    draw();

</script>
</body>
</html>
