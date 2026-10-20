
<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Pixel Hero</title>
<style>
* { box-sizing: border-box; }
body {
  margin: 0;
  background: #101827;
  color: white;
  font-family: Arial, sans-serif;
  text-align: center;
  user-select: none;
}
header { padding: 12px; }
h1 { margin: 4px; color: #54e8ff; }
canvas {
  display: block;
  width: 100%;
  max-width: 900px;
  margin: auto;
  background: #80d9ff;
  border: 3px solid #fff;
  touch-action: none;
}
.controls {
  display: flex;
  justify-content: center;
  gap: 14px;
  padding: 15px;
}
button {
  background: #263b60;
  color: white;
  border: 2px solid #54e8ff;
  border-radius: 14px;
  padding: 15px 23px;
  font-size: 20px;
  font-weight: bold;
  touch-action: none;
}
button:active { background: #087d9a; }
#restart { font-size: 15px; padding: 10px 16px; }
p { margin: 8px; }
</style>
</head>
<body>
<header>
  <h1>PIXEL HERO</h1>
  <p>Kumpulkan koin dan capai garis akhir!</p>
  <p id="info">Koin: 0 | Nyawa: 3</p>
</header>

<canvas id="game" width="800" height="450"></canvas>

<div class="controls">
  <button id="left" aria-label="Kiri">◀</button>
  <button id="jump" aria-label="Lompat">▲</button>
  <button id="right" aria-label="Kanan">▶</button>
</div>
<button id="restart">Main Lagi</button>
<p>PC: tombol panah / WASD dan spasi untuk lompat.</p>

<script>
(() => {
  const canvas = document.getElementById("game");
  const ctx = canvas.getContext("2d");
  const info = document.getElementById("info");

  const worldWidth = 2200;
  const floorY = 390;

  const platforms = [
    {x:0, y:390, w:500, h:60},
    {x:570, y:340, w:180, h:25},
    {x:820, y:290, w:180, h:25},
    {x:1080, y:345, w:200, h:25},
    {x:1360, y:300, w:180, h:25},
    {x:1620, y:350, w:580, h:100}
  ];

  let player, coins, camera, score, lives, ended, won;
  const keys = {left:false, right:false};

  function reset() {
    player = {
      x:40, y:300, w:30, h:40,
      vx:0, vy:0, grounded:false
    };
    coins = [220, 390, 630, 880, 1140, 1420, 1700, 1900]
      .map(x => ({x, y:260 + (x % 3) * 20, got:false}));
    camera = 0;
    score = 0;
    lives = 3;
    ended = false;
    won = false;
    keys.left = false;
    keys.right = false;
    draw();
  }

  function jump() {
    if (!ended && player.grounded) {
      player.vy = -12;
      player.grounded = false;
    }
  }

  function bindHold(id, key) {
    const button = document.getElementById(id);
    button.addEventListener("pointerdown", e => {
      e.preventDefault();
      keys[key] = true;
      button.setPointerCapture(e.pointerId);
    });
    const release = e => {
      e.preventDefault();
      keys[key] = false;
    };
    button.addEventListener("pointerup", release);
    button.addEventListener("pointercancel", release);
    button.addEventListener("lostpointercapture", () => {
      keys[key] = false;
    });
  }

  bindHold("left", "left");
  bindHold("right", "right");
  document.getElementById("jump").addEventListener("pointerdown", e => {
    e.preventDefault();
    jump();
  });
  document.getElementById("restart").addEventListener("click", reset);

  const downKeys = new Set([
    "ArrowLeft", "ArrowRight", "ArrowUp", " ",
    "a", "d", "w", "A", "D", "W"
  ]);

  window.addEventListener("keydown", e => {
    if (downKeys.has(e.key)) e.preventDefault();
    if (e.key === "ArrowLeft" || e.key.toLowerCase() === "a")
      keys.left = true;
    if (e.key === "ArrowRight" || e.key.toLowerCase() === "d")
      keys.right = true;
    if (["ArrowUp", " ", "w", "W"].includes(e.key) && !e.repeat)
      jump();
    if (ended && e.key === "Enter") reset();
  });

  window.addEventListener("keyup", e => {
    if (e.key === "ArrowLeft" || e.key.toLowerCase() === "a")
      keys.left = false;
    if (e.key === "ArrowRight" || e.key.toLowerCase() === "d")
      keys.right = false;
  });

  function overlap(a, b) {
    return a.x < b.x + b.w &&
      a.x + a.w > b.x &&
      a.y < b.y + b.h &&
      a.y + a.h > b.y;
  }

  function update() {
    if (ended) return;

    player.vx = (keys.right ? 4 : 0) - (keys.left ? 4 : 0);
    player.x += player.vx;
    player.x = Math.max(0, Math.min(worldWidth - player.w, player.x));

    const oldBottom = player.y + player.h;
    player.vy += 0.55;
    player.vy = Math.min(player.vy, 13);
    player.y += player.vy;
    player.grounded = false;

    for (const p of platforms) {
      if (player.vy >= 0 &&
          oldBottom <= p.y &&
          player.y + player.h >= p.y &&
          player.x + player.w > p.x &&
          player.x < p.x + p.w) {
        player.y = p.y - player.h;
        player.vy = 0;
        player.grounded = true;
      }
    }

    for (const c of coins) {
      if (!c.got && overlap(player, {x:c.x, y:c.y, w:18, h:18})) {
        c.got = true;
        score++;
      }
    }

    if (player.y > canvas.height + 100) {
      lives--;
      if (lives <= 0) {
        ended = true;
      } else {
        player.x = 40;
        player.y = 250;
        player.vx = 0;
        player.vy = 0;
      }
    }

    if (player.x > 2100) {
      won = true;
      ended = true;
    }

    camera = Math.max(0, Math.min(
      worldWidth - canvas.width,
      player.x - 250
    ));

    info.textContent =
      `Koin: ${score}/${coins.length} | Nyawa: ${lives}`;
  }

  function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Langit dan awan
    ctx.fillStyle = "#80d9ff";
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = "#ffffff";
    for (let i = 0; i < 7; i++) {
      const x = ((i * 160 - camera * 0.3) % 1000 + 1000) % 1000;
      ctx.fillRect(x, 70 + (i % 3) * 35, 55, 18);
      ctx.fillRect(x + 12, 58 + (i % 3) * 35, 30, 18);
    }

    ctx.save();
    ctx.translate(-camera, 0);

    // Tanah dan platform
    for (const p of platforms) {
      ctx.fillStyle = "#80502b";
      ctx.fillRect(p.x, p.y, p.w, p.h);
      ctx.fillStyle = "#37b34a";
      ctx.fillRect(p.x, p.y, p.w, 9);
    }

    // Koin
    for (const c of coins) {
      if (c.got) continue;
      ctx.fillStyle = "#ffcf33";
      ctx.beginPath();
      ctx.arc(c.x + 9, c.y + 9, 9, 0, Math.PI * 2);
      ctx.fill();
      ctx.fillStyle = "#fff2a0";
      ctx.fillRect(c.x + 6, c.y + 3, 4, 11);
    }

    // Karakter
    ctx.fillStyle = "#e84141";
    ctx.fillRect(player.x, player.y, player.w, player.h);
    ctx.fillStyle = "#ffd09b";
    ctx.fillRect(player.x + 5, player.y + 5, 20, 15);
    ctx.fillStyle = "#172033";
    ctx.fillRect(player.x + 18, player.y + 9, 4, 4);
    ctx.fillStyle = "#283c80";
    ctx.fillRect(player.x + 3, player.y + 24, 24, 12);

    // Garis akhir
    ctx.fillStyle = "#222";
    ctx.fillRect(2130, 260, 6, 130);
    ctx.fillStyle = "#ffdf35";
    ctx.fillRect(2136, 260, 55, 32);
    ctx.fillStyle = "#222";
    ctx.fillRect(2155, 260, 15, 16);

    ctx.restore();

    if (ended) {
      ctx.fillStyle = "rgba(0,0,0,0.7)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "#ffffff";
      ctx.textAlign = "center";
      ctx.font = "bold 34px Arial";
      ctx.fillText(
        won ? "KAMU MENANG!" : "GAME OVER",
        canvas.width / 2, 205
      );
      ctx.font = "20px Arial";
      ctx.fillText(
        `Koin terkumpul: ${score}`,
        canvas.width / 2, 245
      );
      ctx.fillText(
        "Tekan Main Lagi untuk mengulang",
        canvas.width / 2, 285
      );
    }
  }

  function loop() {
    update();
    draw();
    requestAnimationFrame(loop);
  }

  reset();
  loop();
})();
</script>
</body>
</html>

