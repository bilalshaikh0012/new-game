<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Man Flying - Avoid Birds with Difficulty!</title>
  <style>
    body { margin: 0; overflow: hidden; background: #62b3ff;}
    canvas { display: block; margin: auto; background: #add8e6; border: 2px solid #333; }
    #settings {
      position: absolute; left: 10px; top: 14px; z-index: 20;
      background: rgba(255,255,255,0.95); padding:12px; border-radius: 13px;
      font-family: sans-serif; box-shadow: 0 2px 10px rgba(0,0,0,0.07);
      display: flex; flex-direction: column; gap: 8px;
    }
    #stats {
      position: absolute; right: 15px; top: 15px; z-index: 20;
      color: #222; font-family: sans-serif; font-size: 1.2em;
      background: rgba(255,255,255,0.85); padding: 7px 14px; border-radius: 10px;
      border: 1px solid #007;
    }
    #gameover {
      position: absolute; left:50%; top:50%; transform:translate(-50%,-50%);
      z-index:30; font-size:2em; background: #fff; padding:2em 3em;
      border-radius:20px; border:3px solid #f23; color:#c00; display:none;
      font-family: sans-serif;
    }
    button { font-size: 1em; padding: 4px 12px; margin: 4px; }
    label { font-weight: bold; }
    select { font-size: 1em; }
  </style>
</head>
<body>
  <div id="settings">
    <label>
      <input type="checkbox" id="collisionToggle" checked>
      Game Over on Collision
    </label>
    <label>
      Difficulty:
      <select id="difficulty">
        <option value="easy">Easy</option>
        <option value="hard">Hard</option>
        <option value="insane">Insane</option>
      </select>
    </label>
  </div>
  <div id="stats">Score: <span id="score">0</span> | Hits: <span id="hits">0</span></div>
  <div id="gameover">Game Over<br><button id="restartBtn">Restart</button></div>
  <canvas id="game" width="400" height="600"></canvas>
  <script>
    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");
    const collisionToggle = document.getElementById("collisionToggle");
    const difficultySelect = document.getElementById("difficulty");
    const scoreSpan = document.getElementById("score");
    const hitsSpan = document.getElementById("hits");
    const gameOverDiv = document.getElementById("gameover");
    const restartBtn = document.getElementById("restartBtn");

    // Game Settings
    let gameOverOnCollision = collisionToggle.checked;
    let difficulty = difficultySelect.value;

    // Difficulty settings
    const DIFFICULTIES = {
      easy: {birdSpeed: 2, birdInterval: 90},
      hard: {birdSpeed: 4, birdInterval: 60},
      insane: {birdSpeed: 7, birdInterval: 40}
    };
    let birdSpeed = DIFFICULTIES[difficulty].birdSpeed;
    let birdInterval = DIFFICULTIES[difficulty].birdInterval;

    // Player (aadmi)
    let player = {x: 200, y: 300, w: 40, h: 40, speed: 6};

    // Birds array: Each bird has x, y, vx, vy, direction
    let birds = [];
    let birdSize = 32;
    let frame = 0;

    // Game State
    let score = 0, hits = 0, dead = false;
    let keys = {};

    // Listen to collisionToggle
    collisionToggle.addEventListener('change', function() {
      gameOverOnCollision = this.checked;
    });

    // Listen to difficulty change
    difficultySelect.addEventListener('change', function() {
      difficulty = this.value;
      birdSpeed = DIFFICULTIES[difficulty].birdSpeed;
      birdInterval = DIFFICULTIES[difficulty].birdInterval;
    });

    // Movement keys
    window.addEventListener('keydown', e => keys[e.key] = true);
    window.addEventListener('keyup', e => keys[e.key] = false);

    // Restart button
    restartBtn.addEventListener('click', resetGame);

    function resetGame() {
      player.x = 200; player.y = 300;
      birds = []; frame = 0; score = 0; hits = 0; dead = false;
      gameOverDiv.style.display = 'none';
      draw();
    }

    function drawPlayer() {
      ctx.save();
      ctx.fillStyle = '#daa520';
      ctx.beginPath();
      ctx.arc(player.x, player.y, player.w/2, 0, Math.PI*2);
      ctx.fill();
      ctx.fillStyle = "#fde2b3";
      ctx.beginPath();
      ctx.arc(player.x, player.y - player.h/2 + 8, player.w/3, 0, Math.PI*2);
      ctx.fill();
      ctx.strokeStyle = "#deb887"; ctx.lineWidth = 5;
      ctx.beginPath(); ctx.moveTo(player.x-18, player.y-5); ctx.lineTo(player.x+18, player.y-5); ctx.stroke();
      ctx.restore();
    }

    function drawBird(b) {
      ctx.save();
      ctx.fillStyle = "#f44";
      ctx.beginPath();
      ctx.ellipse(b.x, b.y, birdSize/2, birdSize/3, 0, 0, Math.PI*2);
      ctx.fill();
      ctx.fillStyle = "#333";
      ctx.beginPath();
      ctx.arc(b.x+7, b.y-7, 3, 0, Math.PI*2);
      ctx.fill();
      ctx.fillStyle="#ff0";
      ctx.beginPath(); ctx.moveTo(b.x+birdSize/2, b.y);
      ctx.lineTo(b.x+birdSize/2+8, b.y-5); ctx.lineTo(b.x+birdSize/2+8, b.y+5); ctx.closePath();
      ctx.fill();
      ctx.restore();
    }

    // Random bird spawn anywhere, fly opposite side (left/right/top/bottom)
    function spawnBird() {
      // Choose a random direction: 0=left->right, 1=right->left, 2=top->bottom, 3=bottom->top
      const dir = Math.floor(Math.random()*4);
      let b = {};
      if (dir===0) { // left -> right
        b.x = -birdSize; b.y = Math.random()*canvas.height;
        b.vx = birdSpeed; b.vy = 0;
      } else if (dir===1) { // right -> left
        b.x = canvas.width+birdSize; b.y = Math.random()*canvas.height;
        b.vx = -birdSpeed; b.vy = 0;
      } else if (dir===2) { // top -> bottom
        b.x = Math.random()*canvas.width; b.y = -birdSize;
        b.vx = 0; b.vy = birdSpeed;
      } else { // bottom -> top
        b.x = Math.random()*canvas.width; b.y = canvas.height+birdSize;
        b.vx = 0; b.vy = -birdSpeed;
      }
      b.dir = dir;
      return b;
    }

    function update() {
      if (dead) return;
      frame++;
      score += 1;

      // Player movement
      if (keys["ArrowUp"] && player.y > player.h/2) player.y -= player.speed;
      if (keys["ArrowDown"] && player.y < canvas.height - player.h/2) player.y += player.speed;
      if (keys["ArrowLeft"] && player.x > player.w/2) player.x -= player.speed;
      if (keys["ArrowRight"] && player.x < canvas.width - player.w/2) player.x += player.speed;

      // Birds movement
      birds.forEach(b => {
        b.x += b.vx; b.y += b.vy;
      });

      // Remove birds gone off screen (whichever direction)
      birds = birds.filter(b => {
        if (b.dir===0 && b.x > canvas.width+birdSize) return false; // left->right
        if (b.dir===1 && b.x < -birdSize) return false; // right->left
        if (b.dir===2 && b.y > canvas.height+birdSize) return false; // top->bottom
        if (b.dir===3 && b.y < -birdSize) return false; // bottom->top
        return true;
      });

      // Add new birds
      if (frame % birdInterval === 0) {
        birds.push(spawnBird());
      }

      // Collision check
      for (
