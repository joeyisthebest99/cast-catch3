<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Fishing Game</title>
  <style>
    body {
      margin: 0;
      font-family: sans-serif;
      background: #87ceeb;
      display: flex;
      flex-direction: column;
      height: 100vh;
    }
    #ui {
      background: #1e1e1e;
      color: #fff;
      padding: 8px;
      display: flex;
      align-items: center;
      gap: 16px;
      font-size: 14px;
    }
    #ui button {
      padding: 6px 10px;
      cursor: pointer;
    }
    #shopPanel {
      background: #222;
      color: #fff;
      padding: 8px;
      display: none;
      flex-direction: column;
      gap: 6px;
      max-width: 260px;
    }
    #shopPanel h3 {
      margin: 0 0 4px;
    }
    #shopPanel button {
      width: 100%;
      text-align: left;
    }
    #log {
      flex: 0 0 auto;
      background: #333;
      color: #eee;
      padding: 6px;
      font-size: 12px;
      height: 80px;
      overflow-y: auto;
    }
    #gameContainer {
      flex: 1 1 auto;
      display: flex;
    }
    canvas {
      flex: 1;
      background: #4fa3d1;
      display: block;
    }
  </style>
</head>
<body>
  <div id="ui">
    <div><strong>Money:</strong> $<span id="money">0</span></div>
    <div><strong>Rod:</strong> <span id="rodName">Basic Spinning Rod</span></div>
    <div><strong>Inventory:</strong> <span id="inventoryCount">0</span> fish</div>
    <button id="castBtn">Cast</button>
    <button id="reelBtn" disabled>Reel</button>
    <button id="sellBtn">Sell All</button>
    <button id="shopBtn">Shop</button>
  </div>

  <div id="gameContainer">
    <canvas id="gameCanvas" width="800" height="400"></canvas>
    <div id="shopPanel">
      <h3>Rod Shop</h3>
      <small>Each rod changes bite chance & average weight.</small>
      <button data-rod="basicSpinning">
        Basic Spinning Rod — $0 (owned)  
        | Bite: normal, Weight: normal
      </button>
      <button data-rod="proSpinning">
        Pro Spinning Rod — $150  
        | Bite: high, Weight: normal
      </button>
      <button data-rod="basicBaitcaster">
        Basic Baitcaster — $200  
        | Bite: normal, Weight: heavy
      </button>
      <button data-rod="proBaitcaster">
        Pro Baitcaster — $400  
        | Bite: high, Weight: heavy
      </button>
      <button id="closeShopBtn">Close Shop</button>
    </div>
  </div>

  <div id="log"></div>

  <script>
    // --- Game State ---
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    const moneyEl = document.getElementById('money');
    const rodNameEl = document.getElementById('rodName');
    const inventoryCountEl = document.getElementById('inventoryCount');
    const logEl = document.getElementById('log');

    const castBtn = document.getElementById('castBtn');
    const reelBtn = document.getElementById('reelBtn');
    const sellBtn = document.getElementById('sellBtn');
    const shopBtn = document.getElementById('shopBtn');
    const shopPanel = document.getElementById('shopPanel');
    const closeShopBtn = document.getElementById('closeShopBtn');

    let money = 0;
    let inventory = []; // {name, weight, value}
    let currentRodKey = 'basicSpinning';
    let isCasting = false;
    let isLineInWater = false;
    let hasBite = false;
    let biteTimer = 0;
    let castX = canvas.width / 2;
    let castY = 80;
    let hookY = castY;
    let hookTargetY = 260;
    let reelProgress = 0;
    let reelDifficulty = 1;

    const rods = {
      basicSpinning: {
        name: 'Basic Spinning Rod',
        biteChance: 0.5,
        avgWeight: 2,
        price: 0
      },
      proSpinning: {
        name: 'Pro Spinning Rod',
        biteChance: 0.75,
        avgWeight: 2,
        price: 150
      },
      basicBaitcaster: {
        name: 'Basic Baitcaster',
        biteChance: 0.5,
        avgWeight: 4,
        price: 200
      },
      proBaitcaster: {
        name: 'Pro Baitcaster',
        biteChance: 0.8,
        avgWeight: 4,
        price: 400
      }
    };

    const fishTable = [
      { name: 'Bluegill', baseWeight: 1, valuePerKg: 5 },
      { name: 'Bass', baseWeight: 2, valuePerKg: 8 },
      { name: 'Catfish', baseWeight: 3, valuePerKg: 10 },
      { name: 'Carp', baseWeight: 2.5, valuePerKg: 6 },
      { name: 'Trout', baseWeight: 1.5, valuePerKg: 9 }
    ];

    function log(msg) {
      const line = document.createElement('div');
      line.textContent = msg;
      logEl.appendChild(line);
      logEl.scrollTop = logEl.scrollHeight;
    }

    function updateUI() {
      moneyEl.textContent = money.toFixed(0);
      rodNameEl.textContent = rods[currentRodKey].name;
      inventoryCountEl.textContent = inventory.length;
    }

    // --- Casting / Reeling Logic ---
    castBtn.addEventListener('click', () => {
      if (isCasting || isLineInWater) return;
      startCast();
    });

    reelBtn.addEventListener('click', () => {
      if (!isLineInWater) return;
      startReel();
    });

    sellBtn.addEventListener('click', () => {
      sellAllFish();
    });

    shopBtn.addEventListener('click', () => {
      shopPanel.style.display = 'flex';
    });

    closeShopBtn.addEventListener('click', () => {
      shopPanel.style.display = 'none';
    });

    shopPanel.addEventListener('click', (e) => {
      const btn = e.target.closest('button[data-rod]');
      if (!btn) return;
      const rodKey = btn.getAttribute('data-rod');
      buyRod(rodKey);
    });

    function startCast() {
      isCasting = true;
      isLineInWater = false;
      hasBite = false;
      reelProgress = 0;
      hookY = castY;
      hookTargetY = 260;
      biteTimer = 0;
      log('You cast your line...');
      castBtn.disabled = true;
      reelBtn.disabled = true;
    }

    function startReel() {
      if (!hasBite) {
        log('You reel in, but nothing was on the line.');
        resetLine();
        return;
      }
      log('You start reeling in the fish!');
      reelDifficulty = 1 + Math.random() * 1.5; // harder fish sometimes
      reelProgress = 0;
    }

    function resetLine() {
      isCasting = false;
      isLineInWater = false;
      hasBite = false;
      castBtn.disabled = false;
      reelBtn.disabled = true;
    }

    function sellAllFish() {
      if (inventory.length === 0) {
        log('No fish to sell.');
        return;
      }
      let total = 0;
      inventory.forEach(f => total += f.value);
      money += total;
      log(`You sold ${inventory.length} fish for $${total.toFixed(2)}.`);
      inventory = [];
      updateUI();
    }

    function buyRod(rodKey) {
      const rod = rods[rodKey];
      if (!rod) return;
      if (rodKey === currentRodKey) {
        log(`You already have the ${rod.name} equipped.`);
        return;
      }
      if (money < rod.price) {
        log(`Not enough money for ${rod.name}. Need $${rod.price}.`);
        return;
      }
      money -= rod.price;
      currentRodKey = rodKey;
      log(`You bought and equipped the ${rod.name}.`);
      updateUI();
    }

    // --- Fish Generation ---
    function rollForBite() {
      const rod = rods[currentRodKey];
      const baseChance = rod.biteChance;
      const rng = Math.random();
      if (rng < baseChance) {
        hasBite = true;
        log('A fish bites! Hit REEL to try to catch it!');
        reelBtn.disabled = false;
      }
    }

    function generateFish() {
      const rod = rods[currentRodKey];
      const fish = fishTable[Math.floor(Math.random() * fishTable.length)];
      const weightVariance = (Math.random() * 0.8 + 0.6); // 0.6–1.4
      const weight = (fish.baseWeight + rod.avgWeight * 0.3) * weightVariance;
      const value = weight * fish.valuePerKg;
      return {
        name: fish.name,
        weight,
        value
      };
    }

    function finishReel(success) {
      if (!success) {
        log('The fish got away...');
        resetLine();
        return;
      }
      const caughtFish = generateFish();
      inventory.push(caughtFish);
      log(`You caught a ${caughtFish.name} weighing ${caughtFish.weight.toFixed(2)} kg worth $${caughtFish.value.toFixed(2)}.`);
      updateUI();
      resetLine();
    }

    // --- Game Loop / Drawing ---
    let lastTime = 0;
    function gameLoop(timestamp) {
      const dt = (timestamp - lastTime) / 1000;
      lastTime = timestamp;
      update(dt);
      draw();
      requestAnimationFrame(gameLoop);
    }

    function update(dt) {
      // Casting animation: drop hook into water
      if (isCasting && !isLineInWater) {
        hookY += 120 * dt;
        if (hookY >= hookTargetY) {
          hookY = hookTargetY;
          isLineInWater = true;
          isCasting = false;
          log('Your line is in the water. Waiting for a bite...');
          // start bite timer
          biteTimer = 0;
        }
      }

      // Bite logic
      if (isLineInWater && !hasBite) {
        biteTimer += dt;
        if (biteTimer > 1.5) {
          // every 1.5s, chance for bite
          biteTimer = 0;
          rollForBite();
        }
      }

      // Reeling minigame: hold reel button repeatedly
      if (hasBite && isLineInWater && reelProgress >= 0) {
        // reelProgress increases when player presses reelBtn
        // We'll simulate tension: each frame, progress decays a bit
        reelProgress -= dt * reelDifficulty * 0.4;
        if (reelProgress < 0) reelProgress = 0;
      }
    }

    function draw() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // Sky
      ctx.fillStyle = '#87ceeb';
      ctx.fillRect(0, 0, canvas.width, 120);

      // Water
      ctx.fillStyle = '#4fa3d1';
      ctx.fillRect(0, 120, canvas.width, canvas.height - 120);

      // Dock
      ctx.fillStyle = '#8b5a2b';
      ctx.fillRect(castX - 60, 90, 120, 30);

      // Rod (simple line)
      ctx.strokeStyle = '#000';
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(castX, 90);
      ctx.lineTo(castX + 40, 60);
      ctx.stroke();

      // Line
      ctx.strokeStyle = '#fff';
      ctx.lineWidth = 1;
      ctx.beginPath();
      ctx.moveTo(castX + 40, 60);
      ctx.lineTo(castX + 40, hookY);
      ctx.stroke();

      // Hook
      ctx.fillStyle = hasBite ? '#ff0000' : '#000000';
      ctx.beginPath();
      ctx.arc(castX + 40, hookY, 5, 0, Math.PI * 2);
      ctx.fill();

      // Reeling bar (if bite)
      if (hasBite) {
        const barWidth = 160;
        const barHeight = 16;
        const barX = 20;
        const barY = 20;
        ctx.fillStyle = '#222';
        ctx.fillRect(barX, barY, barWidth, barHeight);
        ctx.fillStyle = '#0f0';
        const progressWidth = Math.max(0, Math.min(barWidth, reelProgress * 20));
        ctx.fillRect(barX, barY, progressWidth, barHeight);
        ctx.strokeStyle = '#fff';
        ctx.strokeRect(barX, barY, barWidth, barHeight);
        ctx.fillStyle = '#fff';
        ctx.font = '12px sans-serif';
        ctx.fillText('Reel Progress', barX, barY - 4);
      }
    }

    // Reel button mechanic: each click adds progress
    reelBtn.addEventListener('mousedown', () => {
      if (!hasBite || !isLineInWater) return;
      reelProgress += 0.6;
      if (reelProgress >= 8) {
        // success threshold
        finishReel(true);
        reelProgress = -1; // stop bar
      }
    });

    // Start loop
    updateUI();
    requestAnimationFrame(gameLoop);
  </script>
</body>
</html>
