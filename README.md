<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Endless Runner - Slow Mode</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #222;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: 'Courier New', Courier, monospace;
            overflow: hidden;
        }

        #game-container {
            position: relative;
            box-shadow: 0 0 20px rgba(0,0,0,0.8);
        }

        canvas {
            background-color: #87CEEB;
            border: 4px solid #111;
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
            text-shadow: 3px 3px 0 #000;
            user-select: none;
            pointer-events: none;
        }

        .overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.85);
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
            color: #ff4c4c;
            text-shadow: 4px 4px 0 #600;
        }

        .overlay p {
            font-size: 24px;
            margin-bottom: 30px;
        }

        .overlay button {
            padding: 15px 40px;
            font-size: 24px;
            font-family: inherit;
            cursor: pointer;
            border: 4px solid #fff;
            background-color: transparent;
            color: white;
            text-transform: uppercase;
            transition: all 0.2s;
        }

        .overlay button:hover {
            background-color: white;
            color: black;
        }
        
        #start-screen { display: flex; }
        #start-screen h1 { color: #f1c40f; text-shadow: 4px 4px 0 #b7950b; }
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
        <h1>ENDLESS RUNNER</h1>
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
    let gameState = "start"; // start, playing, gameover
    let frameCount = 0;
    
    // Core variables
    let gameSpeed = 4; // SLOWED DOWN: Was 7, now 4
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
    let scenery = { mountains: [], clouds: [], trees: [], groundStripes: [] };

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
        if (gameState === "playing") {
            if (player.grounded) {
                player.dy = player.jumpPower;
                player.grounded = false;
            }
        }
    });

    function initGame() {
        gameSpeed = 4; // SLOWED DOWN: Starting speed
        distance = 0;
        coinsCollected = 0;
        frameCount = 0;
        spawnTimer = 60; 
        
        scoreElement.innerText = coinsCollected;
        distanceElement.innerText = Math.floor(distance);

        // Player (Fixed X position)
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
        
        // Initial Scenery Setup
        scenery.mountains = [
            { x: 0, width: 400, height: 250, color: "#95a5a6" },
            { x: 350, width: 500, height: 350, color: "#7f8c8d" },
            { x: 800, width: 450, height: 300, color: "#95a5a6" }
        ];
        scenery.clouds = [
            { x: 100, y: 100, scale: 1.0 }, { x: 450, y: 60, scale: 1.3 },
            { x: 800, y: 120, scale: 0.9 }, { x: 1100, y: 80, scale: 1.1 }
        ];
        scenery.trees = [
            { x: 200, scale: 1.0 }, { x: 500, scale: 0.8 }, 
            { x: 900, scale: 1.2 }, { x: 1300, scale: 1.1 }
        ];
        scenery.groundStripes = [];
        for (let i = 0; i < 20; i++) scenery.groundStripes.push(i * 60);
    }

    // --- Entity Spawning Logic ---
    function spawnObstacles() {
        spawnTimer--;
        if (spawnTimer <= 0) {
            let rand = Math.random();
            let spawnX = canvas.width + 50;

            if (rand < 0.35) {
                spawnMonster(spawnX, ground.y - 70);
                spawnTimer = Math.random() * 80 + 80; // Wait longer between spawns because game is slower
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
            color: "#2c3e50",
            colorSpikes: "#c0392b",
            // SLOWED DOWN: Monsters move slightly slower to match the player's slower speed
            speedOffset: Math.random() * 1.5 + 0.5 
        });
    }

    function spawnCoins(startX, y, count) {
        for (let i = 0; i < count; i++) {
            coins.push({ x: startX + (i * 40), y: y, radius: 12 });
        }
    }

    // --- Core Update Loop ---
    function update() {
        frameCount++;
        
        // SLOWED DOWN: The rate at which the game speeds up over time is halved
        gameSpeed += 0.001; 
        distance += gameSpeed / 60;
        distanceElement.innerText = Math.floor(distance);

        // Update Player Physics
        player.dy += gravity;
        player.y += player.dy;

        // Player Ground Collision
        player.grounded = false;
        if (player.y + player.height >= ground.y) {
            player.y = ground.y - player.height;
            player.dy = 0;
            player.grounded = true;
        }

        // Platform Collision
        platforms.forEach(plat => {
            if (player.dy > 0 && 
                player.x + player.width > plat.x && 
                player.x < plat.x + plat.width) {
                
                let playerBottom = player.y + player.height;
                let prevPlayerBottom = playerBottom - player.dy;

                if (prevPlayerBottom <= plat.y && playerBottom >= plat.y) {
                    player.y = plat.y - player.height;
                    player.dy = 0;
                    player.grounded = true;
                }
            }
        });

        // Spawn New Entities
        spawnObstacles();

        // Update Environment & Scenery (Scrolling Left)
        platforms.forEach(p => p.x -= gameSpeed);
        coins.forEach(c => c.x -= gameSpeed);
        monsters.forEach(m => m.x -= (gameSpeed + m.speedOffset));
        
        // Parallax Scenery
        scenery.mountains.forEach(m => { m.x -= gameSpeed * 0.2; if (m.x + m.width < 0) m.x = canvas.width + Math.random() * 100; });
        scenery.clouds.forEach(c => { c.x -= gameSpeed * 0.4; if (c.x + 100 < 0) c.x = canvas.width + Math.random() * 200; });
        scenery.trees.forEach(t => { t.x -= gameSpeed; if (t.x + 100 < 0) t.x = canvas.width + Math.random() * 200; });
        for(let i=0; i<scenery.groundStripes.length; i++) {
            scenery.groundStripes[i] -= gameSpeed;
            if (scenery.groundStripes[i] < -50) scenery.groundStripes[i] += canvas.width + 50;
        }

        // Filter out off-screen entities
        platforms = platforms.filter(p => p.x + p.width > -100);
        coins = coins.filter(c => c.x + c.radius > -50);
        monsters = monsters.filter(m => m.x + m.width > -100);

        // Check Collisions - Coins
        for (let i = coins.length - 1; i >= 0; i--) {
            let c = coins[i];
            let coinBox = { x: c.x - c.radius, y: c.y - c.radius, width: c.radius * 2, height: c.radius * 2 };
            if (rectIntersect(player, coinBox)) {
                coinsCollected++;
                scoreElement.innerText = coinsCollected;
                coins.splice(i, 1);
            }
        }

        // Check Collisions - Monsters (Game Over)
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

        let skyGradient = ctx.createLinearGradient(0, 0, 0, ground.y);
        skyGradient.addColorStop(0, "#2980b9");
        skyGradient.addColorStop(1, "#87CEEB");
        ctx.fillStyle = skyGradient;
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        drawMountains();
        drawClouds();
        drawTrees();

        ctx.fillStyle = "#3CB371";
        ctx.fillRect(0, ground.y, canvas.width, ground.height);
        ctx.fillStyle = "#8B4513";
        ctx.fillRect(0, ground.y + 15, canvas.width, ground.height - 15);
        
        ctx.fillStyle = "#654321";
        scenery.groundStripes.forEach(x => {
            ctx.fillRect(x, ground.y + 25, 10, 10);
            ctx.fillRect(x + 20, ground.y + 45, 15, 10);
        });

        platforms.forEach(plat => {
            ctx.fillStyle = "#6e4a33";
            ctx.fillRect(plat.x, plat.y, plat.width, plat.height);
            ctx.strokeStyle = "#4a3121";
            ctx.lineWidth = 3;
            ctx.strokeRect(plat.x, plat.y, plat.width, plat.height);
            ctx.fillStyle = "#55bd7a";
            ctx.fillRect(plat.x, plat.y, plat.width, 5);
        });

        ctx.lineWidth = 2;
        coins.forEach(coin => {
            ctx.fillStyle = "#f1c40f";
            ctx.strokeStyle = "#f39c12";
            ctx.beginPath();
            ctx.arc(coin.x, coin.y, coin.radius, 0, Math.PI * 2);
            ctx.fill(); ctx.stroke();
            ctx.fillStyle = "white";
            ctx.beginPath();
            ctx.arc(coin.x - 4, coin.y - 4, 3, 0, Math.PI * 2);
            ctx.fill();
        });

        monsters.forEach(m => drawMonster(m));
        drawPlayer();
    }

    function drawMountains() {
        scenery.mountains.forEach(mtn => {
            ctx.fillStyle = mtn.color;
            ctx.beginPath();
            ctx.moveTo(mtn.x, ground.y);
            ctx.lineTo(mtn.x + mtn.width / 2, ground.y - mtn.height);
            ctx.lineTo(mtn.x + mtn.width, ground.y);
            ctx.fill();
        });
    }

    function drawClouds() {
        ctx.fillStyle = "white";
        scenery.clouds.forEach(c => {
            ctx.beginPath();
            ctx.arc(c.x, c.y, 20*c.scale, 0, Math.PI*2);
            ctx.arc(c.x+25*c.scale, c.y-10*c.scale, 25*c.scale, 0, Math.PI*2);
            ctx.arc(c.x+50*c.scale, c.y, 20*c.scale, 0, Math.PI*2);
            ctx.arc(c.x+25*c.scale, c.y+10*c.scale, 20*c.scale, 0, Math.PI*2);
            ctx.fill();
        });
    }

    function drawTrees() {
        scenery.trees.forEach(t => {
            ctx.fillStyle = "#a0522d"; 
            ctx.fillRect(t.x - 10*t.scale, ground.y - 40*t.scale, 20*t.scale, 40*t.scale);
            ctx.fillStyle = "#1e7a1e"; 
            ctx.beginPath();
            ctx.moveTo(t.x, ground.y - 90*t.scale);
            ctx.lineTo(t.x - 30*t.scale, ground.y - 40*t.scale);
            ctx.lineTo(t.x + 30*t.scale, ground.y - 40*t.scale);
            ctx.fill();
        });
    }

    function drawMonster(m) {
        // SLOWED DOWN: The monster's bounce animation is slower
        let runBounce = Math.sin(frameCount * 0.25) * 3;
        let my = m.y + runBounce;

        ctx.fillStyle = m.colorSpikes;
        ctx.beginPath();
        ctx.moveTo(m.x, my);
        ctx.lineTo(m.x + m.width/4, my - 20);
        ctx.lineTo(m.x + m.width/2, my);
        ctx.lineTo(m.x + 3*m.width/4, my - 20);
        ctx.lineTo(m.x + m.width, my);
        ctx.fill();

        ctx.fillStyle = m.color;
        ctx.fillRect(m.x, my, m.width, m.height);
        
        ctx.fillStyle = "#e74c3c";
        ctx.beginPath(); ctx.arc(m.x + 15, my + 20, 7, 0, Math.PI*2); ctx.fill();
        ctx.beginPath(); ctx.arc(m.x + 35, my + 20, 7, 0, Math.PI*2); ctx.fill();
        ctx.fillStyle = "black";
        ctx.beginPath(); ctx.arc(m.x + 13, my + 20, 3, 0, Math.PI*2); ctx.fill(); 
        ctx.beginPath(); ctx.arc(m.x + 33, my + 20, 3, 0, Math.PI*2); ctx.fill();

        ctx.fillStyle = "white";
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

        // SLOWED DOWN: The player's running animation is cut in half (from 0.4 to 0.2)
        let legSwing = player.grounded ? Math.sin(frameCount * 0.2) * 12 : 0;
        let armSwing = player.grounded ? Math.sin(frameCount * 0.2) * 10 : -15;
        let bodyBounce = player.grounded ? Math.abs(Math.sin(frameCount * 0.2)) * 3 : 0;
        
        py += bodyBounce; 

        // 1. Back Arm
        ctx.fillStyle = player.colorSkin;
        ctx.fillRect(px + pw/2 - 4 + armSwing, py + 15, 8, 20);

        // 2. Back Leg
        ctx.fillStyle = player.colorPants;
        ctx.fillRect(px + pw/2 - 4 - legSwing, py + ph - 25, 10, 25);

        // 3. Body/Shirt
        ctx.fillStyle = player.colorShirt;
        ctx.fillRect(px, py + ph - 45, pw, 25);

        // 4. Front Leg
        ctx.fillStyle = player.colorPants;
        ctx.fillRect(px + pw/2 - 6 + legSwing, py + ph - 25, 12, 25);

        // 5. Front Arm
        ctx.fillStyle = player.colorSkin;
        ctx.fillRect(px + pw/2 - 4 - armSwing, py + 15, 8, 20);

        // 6. Head & Hair
        ctx.fillStyle = player.colorSkin;
        ctx.fillRect(px + 4, py, pw - 4, 18); 
        ctx.fillStyle = player.colorHair;
        ctx.fillRect(px + 2, py - 4, pw - 2, 8); 
        ctx.fillRect(px + 4, py + 2, 6, 12); 

        // 7. Eyes
        ctx.fillStyle = "white";
        ctx.fillRect(px + pw - 8, py + 4, 6, 6);
        ctx.fillStyle = "black";
        ctx.fillRect(px + pw - 5, py + 6, 3, 3);
    }

    // --- Game Control ---
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
