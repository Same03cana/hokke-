<!DOCTYPE html>
<html>
<head>
  <title>レッツ☆エアホッケー</title>
  <style>
    canvas {
      background: #0a0a0a;
      display: block;
      margin: 10px auto;
      border: 4px solid white;
    }
    body {
      text-align: center;
      background: #111;
      color: white;
      font-family: sans-serif;
    }
    button, select {
      padding: 10px 20px;
      font-size: 16px;
      margin: 10px 5px 0 5px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <h1>レッツ☆エアホッケー</h1>
  <label for="difficulty">難易度：</label>
  <select id="difficulty">
    <option value="easy" selected>Easy</option>
    <option value="hard">Hard</option>
  </select>
  <br/>
  <canvas id="gameCanvas" width="800" height="400"></canvas><br />
  <button onclick="resetGame()">リスタート</button>
  <button id="resetPuckBtn">ボールリセット</button>

  <script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    const difficultySelect = document.getElementById("difficulty");

    const WIDTH = canvas.width;
    const HEIGHT = canvas.height;

    const WINNING_SCORE = 15;

    const paddleRadius = 30;
    let player = { x: 100, y: HEIGHT / 2 };
    let ai = { x: WIDTH - 100, y: HEIGHT / 2 };
    let prevPlayerPos = { x: player.x, y: player.y };

    let targetPlayerPos = { x: player.x, y: player.y };

    // ボール
    let puck = { x: WIDTH / 2, y: HEIGHT / 2, vx: 6, vy: 6, radius: 15 };

    let playerScore = 0;
    let aiScore = 0;

    let beams = [];
    let aiBeams = [];


    let aiFrozen = false;
    let aiFreezeTimer = 0;
    let aiShootCooldown = 0;

    let playerFrozen = false;
    let playerFreezeTimer = 0;

    let playerCanShoot = true;
    let playerCooldown = 0;
    const PLAYER_COOLDOWN_MAX_EASY = 60;
    const PLAYER_COOLDOWN_MAX_HARD = 100;

    let PLAYER_COOLDOWN_MAX = PLAYER_COOLDOWN_MAX_EASY; // 初期はeasy

    // 難易度によるボール速度倍率とカーブ強さ
    const BALL_SPEED_EASY = 6.5;
    const BALL_SPEED_HARD = 9;
    const BALL_CURVE_STRENGTH_HARD = 1.0;

    // マウス操作
    canvas.addEventListener("mousemove", (e) => {
      if (playerFrozen) return;
      const rect = canvas.getBoundingClientRect();
      const mx = e.clientX - rect.left;
      const my = e.clientY - rect.top;

      targetPlayerPos.x = Math.max(paddleRadius, Math.min(mx, WIDTH / 2 - paddleRadius));
      targetPlayerPos.y = Math.max(paddleRadius, Math.min(my, HEIGHT - paddleRadius));
    });

  canvas.addEventListener("click", () => {
  if (playerCanShoot && !playerFrozen) {
    // ★ここを変更します★
    const beamSpeed = difficultySelect.value === "hard" ? 15 : 10; // Hardなら15、Easyなら10
    beams.push({ x: player.x, y: player.y, vx: beamSpeed }); // 変更後: beamSpeed を使う
    playerCanShoot = false;
    playerCooldown = PLAYER_COOLDOWN_MAX;
  }
});

    // 難易度切り替え時の処理
    difficultySelect.addEventListener("change", () => {
      const diff = difficultySelect.value;
      if (diff === "easy") {
        PLAYER_COOLDOWN_MAX = PLAYER_COOLDOWN_MAX_EASY;
        resetPuckSpeed(BALL_SPEED_EASY);
      } else {
        PLAYER_COOLDOWN_MAX = PLAYER_COOLDOWN_MAX_HARD;
        resetPuckSpeed(BALL_SPEED_HARD);
      }
    });

    function resetPuckSpeed(speed) {
      const angle = Math.random() * 2 * Math.PI;
      puck.vx = speed * Math.cos(angle);
      puck.vy = speed * Math.sin(angle);
      puck.x = WIDTH / 2;
      puck.y = HEIGHT / 2;
    }

    function resetPuck() {
      if(difficultySelect.value === "easy"){
        resetPuckSpeed(BALL_SPEED_EASY);
      } else {
        resetPuckSpeed(BALL_SPEED_HARD);
      }
    }

    function resetGame() {
      playerScore = 0;
      aiScore = 0;
      resetPuck();
    }

    document.getElementById("resetPuckBtn").addEventListener("click", () => {
      resetPuck();
    });

    function moveAI() {
      if (aiFrozen) return;
      const speed = 8;
      const noiseY = (Math.random() - 0.5) * 2;

      if (ai.y < puck.y + noiseY) ai.y += speed;
      else if (ai.y > puck.y + noiseY) ai.y -= speed;

      ai.y = Math.max(paddleRadius, Math.min(HEIGHT - paddleRadius, ai.y));
    }

    function updatePlayerPosition() {
      const easing = 0.25;
      player.x += (targetPlayerPos.x - player.x) * easing;
      player.y += (targetPlayerPos.y - player.y) * easing;
    }

    function updatePuck() {
      // 難易度がhardなら微小なカーブを加える
      if(difficultySelect.value === "hard"){
        const curveX = (Math.random() - 0.5) * BALL_CURVE_STRENGTH_HARD;
        const curveY = (Math.random() - 0.5) * BALL_CURVE_STRENGTH_HARD;
        puck.vx += curveX;
        puck.vy += curveY;
      }

      puck.x += puck.vx;
      puck.y += puck.vy;

      if (puck.y < puck.radius || puck.y > HEIGHT - puck.radius) {
        puck.vy *= -1;
      }

      if (puck.x < 0) {
        aiScore++;
        checkWin();
        resetPuck();
      } else if (puck.x > WIDTH) {
        playerScore++;
        checkWin();
        resetPuck();
      }

      [player, ai].forEach(pad => {
        const dx = puck.x - pad.x;
        const dy = puck.y - pad.y;
        const dist = Math.sqrt(dx * dx + dy * dy);
        if (dist < paddleRadius + puck.radius) {
          const angle = Math.atan2(dy, dx);
          let baseSpeed = difficultySelect.value === "hard" ? BALL_SPEED_HARD + 1 : BALL_SPEED_EASY + 1;
          puck.vx = baseSpeed * Math.cos(angle);
          puck.vy = baseSpeed * Math.sin(angle);

          if (pad === player) {
            const vx = player.x - prevPlayerPos.x;
            const vy = player.y - prevPlayerPos.y;
            puck.vx += vx * 0.3;
            puck.vy += vy * 0.3;
          }
        }
      });
    }

    function updateBeams() {
      beams.forEach((beam, i) => {
        beam.x += beam.vx;
        const dx = beam.x - ai.x;
        const dy = beam.y - ai.y;
        const dist = Math.sqrt(dx * dx + dy * dy);
        if (dist < paddleRadius) {
          aiFrozen = true;
          aiFreezeTimer = 120;
          beams.splice(i, 1);
        } else if (beam.x > WIDTH) {
          beams.splice(i, 1);
        }
      });

      if (aiFrozen) {
        aiFreezeTimer--;
        if (aiFreezeTimer <= 0) aiFrozen = false;
      }
    }

    function updateAIBeams() {
      aiBeams.forEach((beam, i) => {
        beam.x += beam.vx;
        const dx = beam.x - player.x;
        const dy = beam.y - player.y;
        const dist = Math.sqrt(dx * dx + dy * dy);
        if (dist < paddleRadius) {
          playerFrozen = true;
          playerFreezeTimer = 120;
          aiBeams.splice(i, 1);
        } else if (beam.x < 0) {
          aiBeams.splice(i, 1);
        }
      });

      if (playerFrozen) {
        playerFreezeTimer--;
        if (playerFreezeTimer <= 0) playerFrozen = false;
      }
    }

function aiShootLogic() {
  // 難易度でクールダウン時間変化
  const baseCooldown = difficultySelect.value === "hard" ? 150 + Math.floor(Math.random() * 100) : 80 + Math.floor(Math.random() * 70);

  if (aiShootCooldown > 0) {
    aiShootCooldown--;
  } else {
    // ★ここを変更します★
    const aiBeamSpeed = difficultySelect.value === "hard" ? -15 : -10; // Hardなら-15、Easyなら-10
    aiBeams.push({ x: ai.x, y: ai.y, vx: aiBeamSpeed }); // 変更後: aiBeamSpeed を使う
    aiShootCooldown = baseCooldown;
  }
}
    function updatePlayerCooldown() {
      if (!playerCanShoot) {
        playerCooldown--;
        if (playerCooldown <= 0) {
          playerCanShoot = true;
        }
      }
    }

    function checkWin(){
      if(playerScore >= WINNING_SCORE){
        alert("あなたの勝ち！");
        resetGame();
      } else if(aiScore >= WINNING_SCORE){
        alert("AIの勝ち！");
        resetGame();
      }
    }

    function drawCircle(x, y, r, color) {
      ctx.fillStyle = color;
      ctx.beginPath();
      ctx.arc(x, y, r, 0, Math.PI * 2);
      ctx.fill();
    }

    function drawBeams() {
      ctx.fillStyle = "lime";
      beams.forEach(beam => {
        ctx.fillRect(beam.x, beam.y - 2, 10, 4);
      });
    }

    function drawAIBeams() {
      ctx.fillStyle = "red";
      aiBeams.forEach(beam => {
        ctx.fillRect(beam.x - 10, beam.y - 2, 10, 4);
      });
    }

    function drawBoundaryLine() {
      ctx.setLineDash([10, 10]);
      ctx.strokeStyle = "white";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(WIDTH / 2, 0);
      ctx.lineTo(WIDTH / 2, HEIGHT);
      ctx.stroke();
      ctx.setLineDash([]);
    }

    function drawCooldownBar() {
      const barWidth = 100;
      const barHeight = 10;
      const x = 20;
      const y = HEIGHT - 30;

      ctx.fillStyle = "gray";
      ctx.fillRect(x, y, barWidth, barHeight);

      const filled = playerCanShoot
        ? barWidth
        : ((PLAYER_COOLDOWN_MAX - playerCooldown) / PLAYER_COOLDOWN_MAX) * barWidth;

      ctx.fillStyle = "lime";
      ctx.fillRect(x, y, filled, barHeight);

      ctx.strokeStyle = "white";
      ctx.strokeRect(x, y, barWidth, barHeight);
    }

    function draw() {
      ctx.clearRect(0, 0, WIDTH, HEIGHT);
      drawBoundaryLine();
      drawBeams();
      drawAIBeams();
      drawCircle(player.x, player.y, paddleRadius, playerFrozen ? "gray" : "blue");
      drawCircle(ai.x, ai.y, paddleRadius, aiFrozen ? "gray" : "red");
      drawCircle(puck.x, puck.y, puck.radius, "white");

      ctx.font = "20px sans-serif";
      ctx.fillText(`You: ${playerScore}`, 100, 30);
      ctx.fillText(`AI: ${aiScore}`, WIDTH - 180, 30);

      drawCooldownBar();
    }

    function gameLoop() {
      aiShootLogic();
      moveAI();
      updatePlayerPosition();
      updatePuck();
      updateBeams();
      updateAIBeams();
      updatePlayerCooldown();
      draw();

      prevPlayerPos.x = player.x;
      prevPlayerPos.y = player.y;

      requestAnimationFrame(gameLoop);
    }

    resetGame();
    gameLoop();
  </script>
</body>
</html>
