# Kanal-K-nig
<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
  <title>Kanal-König – TV-Wagen</title>
  <style>
    :root{
      --bg:#0b0f14;
      --ui:#e8eef7;
      --ui2:#a8b6c9;
      --accent:#7ad7ff;
      --danger:#ff5c7a;
    }
    html,body{height:100%; margin:0; background:var(--bg); color:var(--ui); font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif;}
    .wrap{
      height:100%;
      display:flex;
      flex-direction:column;
      align-items:center;
      justify-content:center;
      gap:10px;
      padding:10px;
      box-sizing:border-box;
    }
    canvas{
      width:min(94vw, 620px);
      height:auto;
      aspect-ratio: 3 / 4;
      border-radius:18px;
      box-shadow: 0 18px 60px rgba(0,0,0,.55);
      background: radial-gradient(120% 120% at 50% 30%, #182436 0%, #0b0f14 65%, #070a0d 100%);
      touch-action:none;
    }
    .hud{
      width:min(94vw, 620px);
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:8px;
      font-size:14px;
      color:var(--ui2);
      user-select:none;
    }
    .hud b{color:var(--ui)}
    .btnrow{
      width:min(94vw, 620px);
      display:flex;
      gap:10px;
      justify-content:space-between;
      user-select:none;
    }
    .btn{
      flex:1;
      padding:14px 12px;
      border-radius:14px;
      border:1px solid rgba(255,255,255,.12);
      background: rgba(255,255,255,.06);
      color:var(--ui);
      font-weight:700;
      letter-spacing:.3px;
      text-align:center;
      -webkit-tap-highlight-color: transparent;
    }
    .btn:active{transform: translateY(1px); background: rgba(255,255,255,.10);}
    .btn.small{
      flex:0 0 auto;
      width:110px;
      font-weight:700;
      opacity:.95;
    }
    .note{
      width:min(94vw, 620px);
      color:rgba(232,238,247,.7);
      font-size:12px;
      line-height:1.3;
      text-align:center;
    }
    .kbd{
      font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
      font-size: 12px;
      padding:2px 6px;
      border-radius:8px;
      border:1px solid rgba(255,255,255,.18);
      background: rgba(255,255,255,.06);
      color: rgba(232,238,247,.9);
    }
  </style>
</head>
<body>
<div class="wrap">
  <div class="hud">
    <div>Score: <b id="score">0</b> · Combo: <b id="combo">x1</b></div>
    <div>Leben: <b id="lives">3</b> · Speed: <b id="spd">1.0</b></div>
  </div>

  <canvas id="c" width="600" height="800" aria-label="Kanal-König Spiel"></canvas>

  <div class="btnrow">
    <div class="btn" id="leftBtn">⬅️ LINKS</div>
    <div class="btn small" id="pauseBtn">⏸︎ Pause</div>
    <div class="btn" id="rightBtn">RECHTS ➡️</div>
  </div>

  <div class="note">
    Steuerung: <span class="kbd">←</span>/<span class="kbd">→</span> oder Touch-Buttons ·
    <span class="kbd">Space</span> Start/Restart · <span class="kbd">P</span> Pause
  </div>
</div>

<script>
(() => {
  const canvas = document.getElementById("c");
  const ctx = canvas.getContext("2d");

  // UI elements
  const scoreEl = document.getElementById("score");
  const comboEl = document.getElementById("combo");
  const livesEl = document.getElementById("lives");
  const spdEl = document.getElementById("spd");
  const leftBtn = document.getElementById("leftBtn");
  const rightBtn = document.getElementById("rightBtn");
  const pauseBtn = document.getElementById("pauseBtn");

  // Helpers
  const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
  const lerp = (a,b,t) => a + (b-a)*t;
  const rand = (a,b) => a + Math.random()*(b-a);

  // Game state
  const W = canvas.width, H = canvas.height;
  const pipe = { cx: W/2, cy: H*0.52, r: Math.min(W,H)*0.42 };

  const player = {
    x: pipe.cx,
    y: pipe.cy + pipe.r*0.55, // bottom-ish inside pipe
    w: 58,
    h: 34,
    vx: 0,
    speed: 420,          // lateral responsiveness
    maxOffset: pipe.r*0.60,
    invuln: 0,           // seconds
    blink: 0,
    ledPulse: 0,         // pulse on pickup
  };

  let keys = { left:false, right:false };
  let touchDir = 0; // -1 left, +1 right, 0 none

  let running = false;
  let paused = false;
  let over = false;

  let score = 0;
  let lives = 3;
  let combo = 1;
  let comboTimer = 0; // seconds before combo resets

  let time = 0;
  let difficulty = 1;
  let baseScroll = 220; // px/s
  let scrollSpeed = baseScroll;

  const obstacles = [];
  const pickups = [];
  const particles = [];

  const OB_TYPES = [
    { id:"wurzeln",  name:"Wurzeln",   kind:"spikes",   w:140, h:70 },
    { id:"einbruch", name:"Einbruch",  kind:"ledge",    w:170, h:55 },
    { id:"fett",     name:"Fettberg",  kind:"blob",     w:160, h:70 },
  ];
  const PU_TYPES = [
    { id:"riss",       name:"Riss",        kind:"crack",   pts: 25 },
    { id:"scherben",   name:"Scherben",    kind:"glass",   pts: 20 },
    { id:"fremdw",     name:"Fremdwasser", kind:"drop",    pts: 30 },
  ];

  function resetGame() {
    obstacles.length = 0;
    pickups.length = 0;
    particles.length = 0;

    score = 0;
    lives = 3;
    combo = 1;
    comboTimer = 0;
    time = 0;
    difficulty = 1;
    scrollSpeed = baseScroll;

    player.x = pipe.cx;
    player.vx = 0;
    player.invuln = 0;
    player.blink = 0;
    player.ledPulse = 0;

    running = true;
    paused = false;
    over = false;

    spawnInitial();
    syncHUD();
  }

  function syncHUD() {
    scoreEl.textContent = Math.floor(score);
    comboEl.textContent = "x" + combo;
    livesEl.textContent = lives;
    spdEl.textContent = (scrollSpeed / baseScroll).toFixed(1);
  }

  function spawnInitial() {
    // start with some spacing
    let y = -120;
    for (let i=0;i<5;i++){
      if (Math.random() < 0.65) spawnObstacle(y);
      if (Math.random() < 0.75) spawnPickup(y - rand(80,160));
      y -= rand(140,220);
    }
  }

  function spawnObstacle(yOverride) {
    const t = OB_TYPES[(Math.random()*OB_TYPES.length)|0];
    const xOff = rand(-player.maxOffset*0.95, player.maxOffset*0.95);
    const o = {
      type: t,
      x: pipe.cx + xOff,
      y: yOverride ?? -80,
      w: t.w * rand(0.8,1.05),
      h: t.h * rand(0.85,1.1),
      rot: rand(-0.12, 0.12),
      // collision radius approximations
      r: 0,
    };
    o.r = Math.max(o.w, o.h)*0.36;
    obstacles.push(o);
  }

  function spawnPickup(yOverride) {
    const t = PU_TYPES[(Math.random()*PU_TYPES.length)|0];
    const xOff = rand(-player.maxOffset*0.95, player.maxOffset*0.95);
    pickups.push({
      type: t,
      x: pipe.cx + xOff,
      y: yOverride ?? -60,
      r: 16,
      bob: rand(0, Math.PI*2),
      taken:false
    });
  }

  function emit(x,y, n, spread=1, speed=220) {
    for (let i=0;i<n;i++){
      const a = rand(-Math.PI, Math.PI) * spread;
      const v = rand(speed*0.4, speed);
      particles.push({
        x,y,
        vx: Math.cos(a)*v,
        vy: Math.sin(a)*v,
        life: rand(0.35, 0.75),
        t: 0,
        s: rand(1.5, 3.8),
      });
    }
  }

  function circleHit(ax,ay,ar, bx,by,br){
    const dx = ax-bx, dy = ay-by;
    return (dx*dx + dy*dy) <= (ar+br)*(ar+br);
  }

  // Input: keyboard
  window.addEventListener("keydown", (e) => {
    if (e.key === "ArrowLeft") keys.left = true;
    if (e.key === "ArrowRight") keys.right = true;

    if (e.key === " "){ // Space
      e.preventDefault();
      if (!running || over) resetGame();
      else if (paused) paused = false;
    }
    if (e.key.toLowerCase() === "p") paused = !paused;
  });
  window.addEventListener("keyup", (e) => {
    if (e.key === "ArrowLeft") keys.left = false;
    if (e.key === "ArrowRight") keys.right = false;
  });

  // Input: touch buttons (also mouse)
  function bindHold(btn, dir){
    let down = false;
    const set = (v)=> { touchDir = v; };
    const onDown = (e)=>{ e.preventDefault(); down = true; set(dir); };
    const onUp = (e)=>{ e.preventDefault(); if (!down) return; down=false; set(0); };

    btn.addEventListener("pointerdown", onDown);
    btn.addEventListener("pointerup", onUp);
    btn.addEventListener("pointercancel", onUp);
    btn.addEventListener("pointerleave", onUp);
  }
  bindHold(leftBtn, -1);
  bindHold(rightBtn, +1);

  pauseBtn.addEventListener("click", () => {
    if (!running && !over) return;
    paused = !paused;
  });

  // Also allow swipe on canvas
  let swipeStartX = null;
  canvas.addEventListener("pointerdown", (e) => {
    canvas.setPointerCapture(e.pointerId);
    swipeStartX = e.clientX;
  });
  canvas.addEventListener("pointermove", (e) => {
    if (swipeStartX == null) return;
    const dx = e.clientX - swipeStartX;
    if (Math.abs(dx) > 10) touchDir = dx < 0 ? -1 : +1;
  });
  canvas.addEventListener("pointerup", (e) => {
    swipeStartX = null;
    touchDir = 0;
  });
  canvas.addEventListener("pointercancel", () => {
    swipeStartX = null;
    touchDir = 0;
  });

  // Game loop
  let last = performance.now();
  function frame(now){
    const dt = Math.min(0.033, (now-last)/1000);
    last = now;

    if (running && !paused && !over) update(dt);
    render(dt);

    requestAnimationFrame(frame);
  }
  requestAnimationFrame(frame);

  function update(dt){
    time += dt;

    // difficulty ramps: smooth increase
    difficulty = 1 + time / 45; // after ~45s -> 2.0
    scrollSpeed = baseScroll * (1 + (difficulty-1)*0.55);

    // Combo decay
    if (comboTimer > 0) comboTimer -= dt;
    else combo = 1;

    // Player movement
    const dir = (keys.left ? -1 : 0) + (keys.right ? +1 : 0) + touchDir;
    const targetV = clamp(dir, -1, +1) * player.speed;
    player.vx = lerp(player.vx, targetV, 0.18);
    player.x += player.vx * dt;

    // constrain to inside pipe
    const off = player.x - pipe.cx;
    player.x = pipe.cx + clamp(off, -player.maxOffset, player.maxOffset);

    // invuln timer
    if (player.invuln > 0) {
      player.invuln -= dt;
      player.blink += dt*18;
    } else {
      player.blink = 0;
    }

    // led pulse
    player.ledPulse = Math.max(0, player.ledPulse - dt*2.2);

    // spawn logic: keep density
    const want = 6;
    while (obstacles.length < want) {
      spawnObstacle(-rand(80, 240));
    }
    while (pickups.length < want) {
      spawnPickup(-rand(80, 260));
    }

    // move obstacles/pickups down
    for (const o of obstacles) o.y += scrollSpeed * dt;
    for (const p of pickups)  p.y += (scrollSpeed*0.96) * dt;

    // cleanup offscreen
    for (let i=obstacles.length-1; i>=0; i--){
      if (obstacles[i].y > H + 140) obstacles.splice(i,1);
    }
    for (let i=pickups.length-1; i>=0; i--){
      if (pickups[i].y > H + 120 || pickups[i].taken) pickups.splice(i,1);
    }

    // collisions
    const pr = 24; // player collision radius
    for (const p of pickups) {
      const bobY = Math.sin(time*4 + p.bob) * 6;
      if (!p.taken && circleHit(player.x, player.y, pr, p.x, p.y + bobY, p.r)) {
        p.taken = true;
        const pts = p.type.pts * combo;
        score += pts;
        combo = clamp(combo + 1, 1, 8);
        comboTimer = 2.2; // keep combo alive
        player.ledPulse = 1.0;
        emit(p.x, p.y, 16, 1, 260);
      }
    }

    if (player.invuln <= 0) {
      for (const o of obstacles) {
        if (circleHit(player.x, player.y, pr, o.x, o.y, o.r)) {
          // hit
          lives -= 1;
          player.invuln = 1.15;
          player.ledPulse = 0;
          combo = 1;
          comboTimer = 0;
          emit(player.x, player.y, 26, 1, 340);
          // knockback
          player.vx += (player.x < o.x ? -1 : 1) * 260;
          break;
        }
      }
      if (lives <= 0) {
        over = true;
        running = true; // still render
      }
    }

    syncHUD();

    // particles
    for (let i=particles.length-1; i>=0; i--){
      const pt = particles[i];
      pt.t += dt;
      pt.x += pt.vx * dt;
      pt.y += pt.vy * dt;
      pt.vx *= (1 - dt*1.2);
      pt.vy *= (1 - dt*1.2);
      if (pt.t >= pt.life) particles.splice(i,1);
    }
  }

  function render(dt){
    ctx.clearRect(0,0,W,H);

    drawPipe();
    drawTrackMarks();
    drawEntities();
    drawPlayer();
    drawVignette();
    drawOverlayText();
  }

  function drawPipe(){
    // Pipe body with layered rings for depth illusion
    const g = ctx.createRadialGradient(pipe.cx, pipe.cy - pipe.r*0.2, pipe.r*0.12, pipe.cx, pipe.cy, pipe.r);
    g.addColorStop(0, "rgba(255,255,255,0.10)");
    g.addColorStop(0.25, "rgba(155,190,255,0.08)");
    g.addColorStop(0.6, "rgba(40,60,90,0.12)");
    g.addColorStop(1, "rgba(0,0,0,0.35)");

    ctx.save();
    ctx.beginPath();
    ctx.arc(pipe.cx, pipe.cy, pipe.r, 0, Math.PI*2);
    ctx.fillStyle = g;
    ctx.fill();

    // inner ring lines
    for (let i=0;i<7;i++){
      const rr = pipe.r * (0.18 + i*0.11);
      ctx.beginPath();
      ctx.arc(pipe.cx, pipe.cy, rr, 0, Math.PI*2);
      ctx.strokeStyle = `rgba(255,255,255,${0.025 + i*0.008})`;
      ctx.lineWidth = 2;
      ctx.stroke();
    }

    // subtle grime patches
    for (let i=0;i<14;i++){
      const a = rand(0, Math.PI*2);
      const rr = rand(pipe.r*0.25, pipe.r*0.95);
      const x = pipe.cx + Math.cos(a)*rr;
      const y = pipe.cy + Math.sin(a)*rr;
      const r = rand(18, 48);
      ctx.beginPath();
      ctx.arc(x,y,r,0,Math.PI*2);
      ctx.fillStyle = `rgba(20,30,45,${rand(0.06,0.12)})`;
      ctx.fill();
    }

    ctx.restore();
  }

  function drawTrackMarks(){
    // moving tick marks in pipe to show forward motion
    const t = time * (scrollSpeed/180);
    ctx.save();
    ctx.beginPath();
    ctx.arc(pipe.cx, pipe.cy, pipe.r*0.98, 0, Math.PI*2);
    ctx.clip();

    for (let i=0;i<18;i++){
      const y = ((i*60 + (t*60)) % (H+120)) - 80;
      const xMid = pipe.cx + Math.sin((y*0.012) + t*0.8) * 18;

      ctx.strokeStyle = "rgba(255,255,255,0.06)";
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(xMid, y);
      ctx.lineTo(xMid, y+26);
      ctx.stroke();
    }
    ctx.restore();
  }

  function drawEntities(){
    // clip to pipe interior
    ctx.save();
    ctx.beginPath();
    ctx.arc(pipe.cx, pipe.cy, pipe.r*0.985, 0, Math.PI*2);
    ctx.clip();

    // obstacles
    for (const o of obstacles) {
      drawObstacle(o);
    }

    // pickups
    for (const p of pickups) {
      drawPickup(p);
    }

    // particles
    for (const pt of particles){
      const a = 1 - (pt.t/pt.life);
      ctx.fillStyle = `rgba(122,215,255,${0.55*a})`;
      ctx.beginPath();
      ctx.arc(pt.x, pt.y, pt.s, 0, Math.PI*2);
      ctx.fill();
    }

    ctx.restore();
  }

  function drawObstacle(o){
    ctx.save();
    ctx.translate(o.x, o.y);
    ctx.rotate(o.rot);

    const kind = o.type.kind;

    if (kind === "spikes") {
      // Wurzeln
      ctx.fillStyle = "rgba(70,40,20,0.95)";
      ctx.strokeStyle = "rgba(15,10,6,0.55)";
      ctx.lineWidth = 2;

      const w = o.w, h = o.h;
      ctx.beginPath();
      ctx.roundRect(-w/2, -h/2, w, h, 14);
      ctx.fill();

      // root tendrils
      const n = 8;
      for (let i=0;i<n;i++){
        const x = lerp(-w*0.45, w*0.45, i/(n-1));
        ctx.beginPath();
        ctx.moveTo(x, -h*0.15);
        ctx.bezierCurveTo(x+rand(-18,18), -h*0.6, x+rand(-28,28), -h*0.9, x+rand(-20,20), -h*1.1);
        ctx.stroke();
      }

      // highlight
      const hg = ctx.createLinearGradient(-w/2, -h/2, w/2, h/2);
      hg.addColorStop(0, "rgba(255,255,255,0.08)");
      hg.addColorStop(1, "rgba(0,0,0,0)");
      ctx.fillStyle = hg;
      ctx.beginPath();
      ctx.roundRect(-w/2, -h/2, w, h, 14);
      ctx.fill();
    }

    if (kind === "ledge") {
      // Einbruch / Versatz
      const w = o.w, h = o.h;
      ctx.fillStyle = "rgba(90,105,130,0.85)";
      ctx.strokeStyle = "rgba(0,0,0,0.35)";
      ctx.lineWidth = 2;

      ctx.beginPath();
      ctx.roundRect(-w/2, -h/2, w, h, 10);
      ctx.fill();
      ctx.stroke();

      // broken edge
      ctx.fillStyle = "rgba(255,255,255,0.10)";
      ctx.beginPath();
      ctx.moveTo(-w*0.5, -h*0.05);
      for (let i=0;i<7;i++){
        ctx.lineTo(-w*0.5 + (i+1)*(w/7), -h*0.05 + rand(-10,10));
      }
      ctx.lineTo(w*0.5, h*0.5);
      ctx.lineTo(-w*0.5, h*0.5);
      ctx.closePath();
      ctx.fill();
    }

    if (kind === "blob") {
      // Fettberg
      const w = o.w, h = o.h;
      const gg = ctx.createRadialGradient(-w*0.15, -h*0.15, 10, 0, 0, Math.max(w,h)*0.7);
      gg.addColorStop(0, "rgba(210,190,120,0.95)");
      gg.addColorStop(1, "rgba(120,110,70,0.95)");
      ctx.fillStyle = gg;
      ctx.strokeStyle = "rgba(30,25,10,0.35)";
      ctx.lineWidth = 2;

      ctx.beginPath();
      const steps = 10;
      for (let i=0;i<=steps;i++){
        const a = (i/steps)*Math.PI*2;
        const rx = (w*0.45) * (0.85 + 0.25*Math.sin(a*3));
        const ry = (h*0.45) * (0.85 + 0.25*Math.cos(a*2));
        const x = Math.cos(a)*rx;
        const y = Math.sin(a)*ry;
        if (i===0) ctx.moveTo(x,y); else ctx.lineTo(x,y);
      }
      ctx.closePath();
      ctx.fill();
      ctx.stroke();

      // oily shine
      ctx.fillStyle = "rgba(255,255,255,0.10)";
      ctx.beginPath();
      ctx.ellipse(-w*0.08, -h*0.18, w*0.18, h*0.12, -0.2, 0, Math.PI*2);
      ctx.fill();
    }

    ctx.restore();
  }

  function drawPickup(p){
    const bobY = Math.sin(time*4 + p.bob) * 6;

    ctx.save();
    ctx.translate(p.x, p.y + bobY);

    // glow
    const gl = ctx.createRadialGradient(0,0, 2, 0,0, 34);
    gl.addColorStop(0, "rgba(122,215,255,0.28)");
    gl.addColorStop(1, "rgba(122,215,255,0)");
    ctx.fillStyle = gl;
    ctx.beginPath();
    ctx.arc(0,0,34,0,Math.PI*2);
    ctx.fill();

    // icon container
    ctx.fillStyle = "rgba(10,18,28,0.65)";
    ctx.strokeStyle = "rgba(122,215,255,0.45)";
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.arc(0,0,p.r+3,0,Math.PI*2);
    ctx.fill();
    ctx.stroke();

    // icon
    ctx.strokeStyle = "rgba(232,238,247,0.85)";
    ctx.lineWidth = 2.2;
    ctx.lineCap = "round";

    const k = p.type.kind;
    if (k==="crack"){
      ctx.beginPath();
      ctx.moveTo(-8,-6); ctx.lineTo(-2,-1); ctx.lineTo(-6,6); ctx.lineTo(2,2); ctx.lineTo(7,8);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(3,-8); ctx.lineTo(1,-1); ctx.lineTo(8,1);
      ctx.stroke();
    } else if (k==="glass"){
      // shards
      ctx.beginPath();
      ctx.moveTo(-8,6); ctx.lineTo(-3,-8); ctx.lineTo(1,0); ctx.closePath();
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(2,7); ctx.lineTo(9,-3); ctx.lineTo(4,-8); ctx.closePath();
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(-9,-1); ctx.lineTo(-2,3); ctx.lineTo(-6,8); ctx.closePath();
      ctx.stroke();
    } else if (k==="drop"){
      // droplet
      ctx.beginPath();
      ctx.moveTo(0,-10);
      ctx.bezierCurveTo(7,-2, 8,4, 0,10);
      ctx.bezierCurveTo(-8,4, -7,-2, 0,-10);
      ctx.stroke();
      ctx.beginPath();
      ctx.arc(0,4,2.2,0,Math.PI*2);
      ctx.stroke();
    }

    ctx.restore();
  }

  function drawPlayer(){
    // clip to pipe
    ctx.save();
    ctx.beginPath();
    ctx.arc(pipe.cx, pipe.cy, pipe.r*0.985, 0, Math.PI*2);
    ctx.clip();

    // LED light cone (in Fahrtrichtung = nach oben)
    const aimX = player.x;
    const aimY = player.y - 260;

    const coneW = 200 + player.ledPulse*90;
    const flicker = 0.85 + 0.15*Math.sin(time*18) + 0.10*Math.sin(time*7.2);
    const intensity = (0.11 + player.ledPulse*0.10) * flicker;

    ctx.save();
    // gradient cone
    const lg = ctx.createRadialGradient(player.x, player.y-20, 18, player.x, player.y-220, 260);
    lg.addColorStop(0, `rgba(122,215,255,${0.10 + intensity})`);
    lg.addColorStop(0.55, `rgba(122,215,255,${0.06 + intensity*0.6})`);
    lg.addColorStop(1, "rgba(122,215,255,0)");
    ctx.fillStyle = lg;

    ctx.beginPath();
    ctx.moveTo(player.x, player.y-10);
    ctx.quadraticCurveTo(player.x - coneW*0.35, player.y-120, player.x - coneW*0.18, aimY);
    ctx.quadraticCurveTo(player.x, aimY-10, player.x + coneW*0.18, aimY);
    ctx.quadraticCurveTo(player.x + coneW*0.35, player.y-120, player.x, player.y-10);
    ctx.closePath();
    ctx.fill();

    // subtle beam edge
    ctx.strokeStyle = `rgba(122,215,255,${0.10 + intensity*0.8})`;
    ctx.lineWidth = 1.5;
    ctx.stroke();

    ctx.restore();

    // TV-Wagen body
    const blinkOn = player.invuln > 0 ? (Math.floor(player.blink)%2===0) : true;
    if (blinkOn) {
      // body shadow
      ctx.fillStyle = "rgba(0,0,0,0.35)";
      ctx.beginPath();
      ctx.ellipse(player.x, player.y+16, 42, 10, 0, 0, Math.PI*2);
      ctx.fill();

      // chassis
      ctx.save();
      ctx.translate(player.x, player.y);

      const tilt = clamp(player.vx/700, -0.22, 0.22);
      ctx.rotate(tilt);

      // base
      ctx.fillStyle = "rgba(220,230,245,0.92)";
      ctx.strokeStyle = "rgba(0,0,0,0.25)";
      ctx.lineWidth = 2;

      ctx.beginPath();
      ctx.roundRect(-player.w/2, -player.h/2, player.w, player.h, 10);
      ctx.fill();
      ctx.stroke();

      // darker top plate
      const topg = ctx.createLinearGradient(0, -player.h/2, 0, player.h/2);
      topg.addColorStop(0, "rgba(20,30,45,0.40)");
      topg.addColorStop(1, "rgba(0,0,0,0.10)");
      ctx.fillStyle = topg;
      ctx.beginPath();
      ctx.roundRect(-player.w/2, -player.h/2, player.w, player.h, 10);
      ctx.fill();

      // wheels
      ctx.fillStyle = "rgba(30,35,45,0.95)";
      ctx.beginPath(); ctx.roundRect(-player.w*0.40, player.h*0.15, 16, 10, 5); ctx.fill();
      ctx.beginPath(); ctx.roundRect(player.w*0.24, player.h*0.15, 16, 10, 5); ctx.fill();

      // camera head (front = top side in our view)
      const camX = 0;
      const camY = -player.h*0.60;
      ctx.fillStyle = "rgba(235,245,255,0.95)";
      ctx.strokeStyle = "rgba(0,0,0,0.25)";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.roundRect(-18, camY-16, 36, 26, 10);
      ctx.fill();
      ctx.stroke();

      // lens
      ctx.fillStyle = "rgba(10,18,28,0.9)";
      ctx.beginPath();
      ctx.arc(0, camY-3, 8.5, 0, Math.PI*2);
      ctx.fill();

      // LED ring + dot
      const ledGlow = ctx.createRadialGradient(0, camY-3, 1, 0, camY-3, 26);
      ledGlow.addColorStop(0, `rgba(122,215,255,${0.40 + player.ledPulse*0.40})`);
      ledGlow.addColorStop(1, "rgba(122,215,255,0)");
      ctx.fillStyle = ledGlow;
      ctx.beginPath();
      ctx.arc(0, camY-3, 26, 0, Math.PI*2);
      ctx.fill();

      ctx.strokeStyle = `rgba(122,215,255,${0.75 + player.ledPulse*0.25})`;
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.arc(0, camY-3, 13.5, 0, Math.PI*2);
      ctx.stroke();

      ctx.fillStyle = `rgba(122,215,255,${0.85 + player.ledPulse*0.15})`;
      ctx.beginPath();
      ctx.arc(10, camY-10, 2.2, 0, Math.PI*2);
      ctx.fill();

      // label
      ctx.fillStyle = "rgba(10,18,28,0.80)";
      ctx.font = "700 10px system-ui, sans-serif";
      ctx.textAlign = "center";
      ctx.fillText("TV", 0, 4);

      ctx.restore();
    }

    ctx.restore();
  }

  function drawVignette(){
    // vignette overlay
    const vg = ctx.createRadialGradient(W/2, H*0.42, H*0.15, W/2, H*0.45, H*0.72);
    vg.addColorStop(0, "rgba(0,0,0,0)");
    vg.addColorStop(1, "rgba(0,0,0,0.55)");
    ctx.fillStyle = vg;
    ctx.fillRect(0,0,W,H);
  }

  function drawOverlayText(){
    if (!running && !over) {
      // not used: we always start via Space or resetGame
      return;
    }

    // Start message if not yet started
    if (!running) return;

    // Pause
    if (paused && !over) {
      panel("PAUSE", "Drück P oder tippe Pause, um weiterzumachen.");
    }

    // Game Over
    if (over) {
      panel("GAME OVER",
        `Score: ${Math.floor(score)}  ·  (Space) Neustart`);
    } else if (time < 2.5 && score === 0) {
      // small intro hint
      hint("Sammle Befunde (Riss/Scherben/Fremdwasser) und weich Hindernissen aus.");
    }
  }

  function panel(title, subtitle){
    ctx.save();
    ctx.fillStyle = "rgba(0,0,0,0.45)";
    ctx.fillRect(0,0,W,H);

    // card
    const cw = W*0.78, ch = 160;
    const x = (W-cw)/2, y = H*0.32;
    ctx.fillStyle = "rgba(15,20,30,0.78)";
    ctx.strokeStyle = "rgba(122,215,255,0.35)";
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.roundRect(x,y,cw,ch,18);
    ctx.fill();
    ctx.stroke();

    ctx.fillStyle = "rgba(232,238,247,0.95)";
    ctx.textAlign = "center";
    ctx.font = "900 30px system-ui, sans-serif";
    ctx.fillText(title, W/2, y+62);

    ctx.fillStyle = "rgba(232,238,247,0.80)";
    ctx.font = "600 14px system-ui, sans-serif";
    ctx.fillText(subtitle, W/2, y+104);

    ctx.fillStyle = "rgba(168,182,201,0.75)";
    ctx.font = "600 12px system-ui, sans-serif";
    ctx.fillText("←/→ oder Touch · Space: Start/Restart · P: Pause", W/2, y+134);

    ctx.restore();
  }

  function hint(text){
    ctx.save();
    const pad = 12;
    ctx.font = "700 12px system-ui, sans-serif";
    const w = ctx.measureText(text).width + pad*2;
    const h = 34;
    const x = (W-w)/2;
    const y = H*0.08;

    ctx.fillStyle = "rgba(15,20,30,0.70)";
    ctx.strokeStyle = "rgba(255,255,255,0.10)";
    ctx.lineWidth = 1.5;
    ctx.beginPath();
    ctx.roundRect(x,y,w,h,12);
    ctx.fill();
    ctx.stroke();

    ctx.fillStyle = "rgba(232,238,247,0.85)";
    ctx.textAlign = "center";
    ctx.fillText(text, W/2, y+22);
    ctx.restore();
  }

  // Polyfill-ish for roundRect (older browsers)
  if (!CanvasRenderingContext2D.prototype.roundRect) {
    CanvasRenderingContext2D.prototype.roundRect = function(x, y, w, h, r) {
      r = Math.min(r, w/2, h/2);
      this.beginPath();
      this.moveTo(x+r, y);
      this.arcTo(x+w, y, x+w, y+h, r);
      this.arcTo(x+w, y+h, x, y+h, r);
      this.arcTo(x, y+h, x, y, r);
      this.arcTo(x, y, x+w, y, r);
      this.closePath();
      return this;
    };
  }

  // Start in "title screen" state
  running = true;
  paused = true; // start paused so user sees instructions
  over = false;
  panel("KANAL-KÖNIG", "Drück Space oder tippe Pause, um zu starten.");
  syncHUD();

  // If user taps pause on first screen -> start game
  pauseBtn.addEventListener("click", () => {
    if (paused && score === 0 && time === 0 && obstacles.length===0 && pickups.length===0) {
      resetGame();
    }
  });

  // Also start if user taps canvas
  canvas.addEventListener("click", () => {
    if (paused && score === 0 && time === 0 && obstacles.length===0 && pickups.length===0) {
      resetGame();
    }
  });

})();
</script>
</body>
</html>
