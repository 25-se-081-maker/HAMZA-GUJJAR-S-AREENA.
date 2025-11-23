<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>HAMZA GUJJAR'S ARENA — Mobile Portrait</title>
  <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
  <style>
    :root {
      --bg:#0b1220; --ui:#121a2b; --neon:#00e0ff; --pink:#ff3b81; --blue:#008bff;
      --text:#e8f0ff; --good:#3cff92; --bad:#ff5a5a;
    }
    html, body {
      margin:0; padding:0; height:100%; width:100%;
      background:var(--bg); color:var(--text);
      font-family:Impact, Arial Black, sans-serif;
      overflow:hidden;
    }
    #stage {
      position:relative; width:100vw; height:100vh;
      background:var(--bg); touch-action:none;
    }
    canvas { width:100%; height:100%; display:block; }

    /* HUD */
    .hud-left {
      position:absolute; top:10px; left:10px;
      display:flex; flex-direction:column; gap:6px; z-index:10;
    }
    .hud-right {
      position:absolute; top:10px; right:10px;
      display:flex; flex-direction:column; align-items:flex-end; gap:6px; z-index:10;
    }
    .badge {
      background:var(--ui); padding:8px 12px; border-radius:10px; font-weight:800;
      box-shadow:0 6px 14px rgba(0,0,0,0.25);
    }
    .bar { width:140px; height:12px; background:#333; border-radius:8px; overflow:hidden; margin-top:6px; }
    .bar span { display:block; height:100%; background:var(--good); width:100%; transition:width .2s; }
    #musicToggle {
      background:linear-gradient(180deg,var(--neon),var(--blue)); color:#06101f;
      border:none; border-radius:10px; padding:8px 12px; font-weight:800; cursor:pointer;
    }

    /* Crosshair */
    .crosshair {
      position:absolute; width:22px; height:22px; border:2px solid var(--neon);
      border-radius:50%; transform:translate(-50%,-50%); pointer-events:none; z-index:5;
    }

    /* Mobile controls */
    .stick, .fire {
      position:absolute; background:rgba(0,0,0,0.28); border:1px solid #555; border-radius:50%;
      display:flex; align-items:center; justify-content:center; color:#fff; font-weight:900; user-select:none; z-index:9;
    }
    .stick { width:130px; height:130px; bottom:20px; left:20px; }
    .fire  { width:94px; height:94px;  bottom:30px; right:20px; }

    /* Start overlay */
    .overlay {
      position:absolute; inset:0; display:flex; align-items:center; justify-content:center;
      background:rgba(0,0,0,0.55); color:#fff; font-weight:900; font-size:22px; z-index:20;
      text-align:center; padding:0 24px;
    }
  </style>
</head>
<body>
  <audio id="bgMusic" preload="auto" loop>
    <source id="bgSource" src="" type="audio/mpeg">
  </audio>

  <div id="stage">
    <canvas id="game"></canvas>
    <div class="crosshair" id="crosshair"></div>

    <div class="hud-left">
      <div class="badge">Score <span id="score">0</span></div>
      <div class="badge">Level <span id="level">1</span></div>
    </div>

    <div class="hud-right">
      <div class="badge">
        <div>Health <span id="hpLabel">100</span></div>
        <div class="bar"><span id="hpBar"></span></div>
      </div>
      <button id="musicToggle" type="button">Mute</button>
    </div>

    <div class="stick" id="stick">Move</div>
    <div class="fire" id="fireBtn">Shoot</div>

    <div class="overlay" id="startOverlay">
      Tap anywhere to start — Aim by dragging, Shoot to fire, Move to shift. Pause: P
    </div>
  </div>

  <script>
  (function(){
    "use strict";

    /* CONFIG */
    var TARGET_IMAGE = "hamza.png";        // replace with your image filename
    var MUSIC_FILE  = "arena-theme.mp3";   // replace with your MP3 filename
    var USE_DATA_URL = false;              // set true to use DATA_URL instead of TARGET_IMAGE
    var DATA_URL = "data:image/svg+xml;utf8," + encodeURIComponent(
      "<svg xmlns='http://www.w3.org/2000/svg' width='128' height='128'>" +
      "<rect width='128' height='128' fill='#1e2a44'/>" +
      "<circle cx='64' cy='64' r='50' fill='none' stroke='#6cc3ff' stroke-width='4'/>" +
      "<text x='64' y='72' font-family='Arial' font-size='20' fill='#e8f0ff' text-anchor='middle'>HAMZA</text>" +
      "</svg>"
    );

    /* Elements */
    var canvas = document.getElementById("game");
    var ctx = canvas.getContext("2d");
    var stage = document.getElementById("stage");
    var crosshair = document.getElementById("crosshair");
    var scoreEl = document.getElementById("score");
    var levelEl = document.getElementById("level");
    var hpLabel = document.getElementById("hpLabel");
    var hpBar = document.getElementById("hpBar");
    var stick = document.getElementById("stick");
    var fireBtn = document.getElementById("fireBtn");
    var startOverlay = document.getElementById("startOverlay");
    var bgMusic = document.getElementById("bgMusic");
    var bgSource = document.getElementById("bgSource");
    var musicToggle = document.getElementById("musicToggle");

    /* Music */
    bgSource.src = MUSIC_FILE;
    bgMusic.load();
    bgMusic.volume = 1.0;
    musicToggle.addEventListener("click", function(){
      bgMusic.muted = !bgMusic.muted;
      musicToggle.textContent = bgMusic.muted ? "Unmute" : "Mute";
    });

    /* Prevent page scroll while interacting in stage */
    function preventDefault(e){ e.preventDefault(); }
    stage.addEventListener("touchstart", preventDefault, { passive: false });
    stage.addEventListener("touchmove", preventDefault, { passive: false });
    stage.addEventListener("touchend", preventDefault, { passive: false });

    /* Sizing */
    var W = 0, H = 0;
    function resize(){
      W = window.innerWidth;
      H = window.innerHeight;
      canvas.width = W;
      canvas.height = H;
    }
    resize();
    window.addEventListener("resize", resize);

    /* State */
    var state = {
      running: false,
      paused: false,
      score: 0,
      level: 1,
      hp: 100,
      maxHp: 100,
      targets: [],
      bullets: [],
      particles: [],
      pistol: { x: W/2, y: H-80, angle: -Math.PI/2, recoil: 0 },
      time: 0
    };
    var keys = {};
    var fireCD = 0;

    /* Image target */
    var targetImg = new Image();
    targetImg.src = USE_DATA_URL ? DATA_URL : TARGET_IMAGE;
    var imageReady = false;
    targetImg.onload = function(){ imageReady = true; };

    /* Start/reset */
    function startGame(){
      state.running = true;
      state.paused = false;
      state.score = 0;
      state.level = 1;
      state.hp = 100;
      state.targets = [];
      state.bullets = [];
      state.particles = [];
      state.pistol = { x: W/2, y: H-80, angle: -Math.PI/2, recoil: 0 };
      spawnLevel(1);
      updateHUD();
      startOverlay.style.display = "none";
      bgMusic.play().catch(function(){});
      loop();
    }
    stage.addEventListener("click", function(){
      if(!state.running){ startGame(); }
    });

    /* Levels */
    function spawnLevel(level){
      state.targets.length = 0;
      var capped = Math.min(level, 100);
      var count = 6 + capped * 2;
      var speed = 1 + capped * 0.05;
      var hp = 1 + Math.floor(capped / 10);

      if(capped >= 100){
        state.targets.push({
          x: W/2, y: -100, w: 96, h: 96, speed: speed * 0.8, hp: 30, boss: true
        });
      }
      var i;
      for(i = 0; i < count; i++){
        state.targets.push({
          x: Math.random() * W,
          y: -40 - Math.random() * 140,
          w: 72, h: 72,
          speed: speed,
          hp: hp,
          boss: false
        });
      }
    }

    /* Keyboard (desktop) */
    window.addEventListener("keydown", function(e){
      keys[e.code] = true;
      if(e.code === "KeyP"){ togglePause(); }
      if(e.code === "Space"){ dash(); }
    });
    window.addEventListener("keyup", function(e){
      keys[e.code] = false;
    });

    /* Aim */
    function setAim(clientX, clientY){
      var r = stage.getBoundingClientRect();
      var x = clientX - r.left;
      var y = clientY - r.top;
      crosshair.style.left = String(x) + "px";
      crosshair.style.top = String(y) + "px";
      state.pistol.angle = Math.atan2(y - state.pistol.y, x - state.pistol.x);
    }
    stage.addEventListener("mousemove", function(e){
      setAim(e.clientX, e.clientY);
    });
    stage.addEventListener("touchstart", function(e){
      var t = e.touches[0];
      setAim(t.clientX, t.clientY);
    }, { passive: false });
    stage.addEventListener("touchmove", function(e){
      var t = e.touches[0];
      setAim(t.clientX, t.clientY);
    }, { passive: false });

    /* Move joystick */
    var stickActive = false;
    var stickCenterX = 0;
    stick.addEventListener("touchstart", function(){
      var r = stick.getBoundingClientRect();
      stickCenterX = r.left + (r.width / 2);
      stickActive = true;
    }, { passive: false });
    stick.addEventListener("touchmove", function(e){
      if(!stickActive){ return; }
      var x = e.touches[0].clientX;
      var dx = x - stickCenterX;
      keys.KeyA = dx < -20;
      keys.KeyD = dx > 20;
    }, { passive: false });
    stick.addEventListener("touchend", function(){
      stickActive = false;
      keys.KeyA = false;
      keys.KeyD = false;
    }, { passive: false });

    /* Fire */
    fireBtn.addEventListener("touchstart", function(e){
      e.preventDefault();
      shoot(true);
    }, { passive: false });
    stage.addEventListener("mousedown", function(){
      shoot(true);
    });

    /* Actions */
    function shoot(burst){
      if(!state.running || state.paused){ return; }
      if(!burst && fireCD > 0){ return; }
      fireCD = 2;
      var ang = state.pistol.angle;
      var s = 9;
      state.bullets.push({
        x: state.pistol.x + Math.cos(ang) * 14,
        y: state.pistol.y + Math.sin(ang) * 14,
        vx: Math.cos(ang) * s,
        vy: Math.sin(ang) * s,
        life: 90
      });
      state.pistol.recoil = 6;
      var i;
      for(i = 0; i < 8; i++){
        state.particles.push({
          x: state.pistol.x, y: state.pistol.y,
          vx: Math.cos(ang) * (Math.random() * 2 + 1),
          vy: Math.sin(ang) * (Math.random() * 2 + 1),
          life: 12, color: "#00e0ff"
        });
      }
    }

    function dash(){
      if(!state.running || state.paused){ return; }
      var dx = Math.cos(state.pistol.angle);
      state.pistol.x = clamp(state.pistol.x + dx * 26, 24, W - 24);
    }

    function togglePause(){
      if(!state.running){ return; }
      state.paused = !state.paused;
      if(!state.paused){ loop(); }
      drawOverlay(state.paused ? "Paused (Press P)" : "");
    }

    /* HUD */
    function updateHUD(){
      scoreEl.textContent = String(state.score);
      levelEl.textContent = String(state.level);
      hpLabel.textContent = String(Math.max(0, Math.floor(state.hp)));
      var pct = Math.max(0, (state.hp / state.maxHp) * 100);
      hpBar.style.width = String(pct) + "%";
      hpBar.style.background = state.hp > 40 ?
        "linear-gradient(90deg,#3cff92,#00ffa7)" :
        "linear-gradient(90deg,#ff9c5a,#ff5a5a)";
    }

    /* Loop */
    function loop(){
      if(!state.running){ draw(); return; }
      if(!state.paused){
        update();
        draw();
        window.requestAnimationFrame(loop);
      } else {
        draw();
      }
    }

    /* Update */
    function update(){
      state.time++;
      if(fireCD > 0){ fireCD--; }
      if(state.pistol.recoil > 0){ state.pistol.recoil--; }

      /* Movement */
      var ax = 0;
      if(keys.KeyA){ ax -= 1; }
      if(keys.KeyD){ ax += 1; }
      state.pistol.x = clamp(state.pistol.x + ax * 3.2, 24, W - 24);

      /* Targets */
      var i;
      for(i = state.targets.length - 1; i >= 0; i--){
        var f = state.targets[i];
        var sway = Math.sin((state.time + i * 23) * 0.02) * (f.boss ? 1.1 : 1.8);
        f.x = clamp(f.x + sway, 20, W - 20);
        f.y += f.speed;
        if(f.y > H + Math.max(f.h, 40)){
          state.hp -= f.boss ? 12 : 5;
          state.targets.splice(i, 1);
        }
      }

      /* Bullets */
      for(i = state.bullets.length - 1; i >= 0; i--){
        var b = state.bullets[i];
        b.x += b.vx;
        b.y += b.vy;
        b.life--;
        if(b.life <= 0 || b.x < -10 || b.x > W + 10 || b.y < -10 || b.y > H + 10){
          state.bullets.splice(i, 1);
        }
      }

      /* Collisions */
      var j;
      for(i = state.targets.length - 1; i >= 0; i--){
        var tf = state.targets[i];
        var cx = tf.x;
        var cy = tf.y;
        var rad = Math.max(tf.w, tf.h) * 0.4;
        for(j = state.bullets.length - 1; j >= 0; j--){
          var tb = state.bullets[j];
          if(Math.hypot(tb.x - cx, tb.y - cy) < (rad + 6)){
            tf.hp -= 1;
            state.bullets.splice(j, 1);
            state.score += tf.boss ? 20 : 10;

            var k;
            var count = tf.boss ? 10 : 7;
            for(k = 0; k < count; k++){
              state.particles.push({
                x: cx, y: cy,
                vx: (Math.random() - 0.5) * (tf.boss ? 5 : 3),
                vy: (Math.random() - 0.5) * (tf.boss ? 5 : 3),
                life: tf.boss ? 24 : 16,
                color: tf.boss ? "#ff6ca8" : "#6cc3ff"
              });
            }
            if(tf.hp <= 0){
              state.targets.splice(i, 1);
              break;
            }
          }
        }
      }

      /* Particles */
      for(i = state.particles.length - 1; i >= 0; i--){
        var p = state.particles[i];
        p.x += p.vx;
        p.y += p.vy;
        p.life--;
        if(p.life <= 0){
          state.particles.splice(i, 1);
        }
      }

      /* Level progression */
      if(state.targets.length === 0){
        if(state.level < 100){
          state.level++;
          state.hp = Math.min(state.maxHp, state.hp + 10);
          spawnLevel(state.level);
        } else {
          state.running = false; /* victory */
        }
      }

      /* Game over */
      if(state.hp <= 0){
        state.hp = 0;
        state.running = false;
      }

      updateHUD();
    }

    /* Draw */
    function draw(){
      ctx.clearRect(0, 0, W, H);

      /* Background grid */
      var x;
      var y;
      ctx.save();
      ctx.globalAlpha = 0.08;
      for(x = 0; x < W; x += 40){
        ctx.beginPath();
        ctx.moveTo(x, 0);
        ctx.lineTo(x, H);
        ctx.strokeStyle = "#6cc3ff";
        ctx.stroke();
      }
      for(y = 0; y < H; y += 40){
        ctx.beginPath();
        ctx.moveTo(0, y);
        ctx.lineTo(W, y);
        ctx.strokeStyle = "#ff6ca8";
        ctx.stroke();
      }
      ctx.restore();

      /* Pistol */
      var ang = state.pistol.angle;
      var px = state.pistol.x;
      var py = state.pistol.y;
      var recoilOffset = state.pistol.recoil * 0.6;
      ctx.save();
      ctx.translate(px, py);
      ctx.rotate(ang);
      ctx.fillStyle = "#00e0ff";
      ctx.strokeStyle = "#008bff";
      ctx.lineWidth = 2;
      ctx.fillRect(-8, -6, 16, 12);
      ctx.fillRect(8 - recoilOffset, -3, 18, 6);
      ctx.strokeRect(8 - recoilOffset, -3, 18, 6);
      ctx.restore();

      /* Targets */
      var i;
      for(i = 0; i < state.targets.length; i++){
        var f = state.targets[i];
        var w = f.w;
        var h = f.h;
        var dx = f.x - w / 2;
        var dy = f.y - h / 2;
        if(imageReady){
          ctx.drawImage(targetImg, dx, dy, w, h);
        } else {
          ctx.fillStyle = "#1e2a44";
          ctx.fillRect(dx, dy, w, h);
        }
        ctx.beginPath();
        ctx.arc(f.x, f.y, Math.max(w, h) * 0.5, 0, Math.PI * 2);
        ctx.strokeStyle = f.boss ? "#ff6ca8" : "#6cc3ff";
        ctx.lineWidth = f.boss ? 3 : 2;
        ctx.stroke();
      }

      /* Bullets */
      for(i = 0; i < state.bullets.length; i++){
        var b = state.bullets[i];
        ctx.beginPath();
        ctx.arc(b.x, b.y, 3, 0, Math.PI * 2);
        ctx.fillStyle = "#e8f0ff";
        ctx.fill();
      }

      /* Particles */
      for(i = 0; i < state.particles.length; i++){
        var p = state.particles[i];
        ctx.beginPath();
        ctx.arc(p.x, p.y, 2, 0, Math.PI * 2);
        ctx.fillStyle = p.color;
        ctx.fill();
      }

      if(state.paused){ drawOverlay("Paused (Press P)"); }
      else if(!state.running && state.hp <= 0){ drawOverlay("Game Over — Tap to Restart"); }
      else if(!state.running && state.level >= 100){ drawOverlay("Champion of HAMZA GUJJAR'S ARENA!"); }
    }

    function drawOverlay(text){
      ctx.save();
      ctx.fillStyle = "rgba(0,0,0,0.45)";
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = "#ffffff";
      ctx.textAlign = "center";
      ctx.font = "bold 26px Arial";
      ctx.fillText(text, W / 2, H / 2);
      ctx.restore();
    }

    function clamp(v, min, max){
      if(v < min){ return min; }
      if(v > max){ return max; }
      return v;
    }
  }());
  </script>
</body>
</html>
