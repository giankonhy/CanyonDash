# CanyonDash

Juego HTML5 estilo *endless runner* con monedas y tienda.

## 📥 Cómo descargar el proyecto

```bash
git clone https://github.com/giankonhy/CanyonDash.git
<!doctype html>
<html lang="es">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Canyon Dash — Demo</title>
<style>
  :root{--bg:#0b1220;--fg:#e6f1ff;--accent:#ffb86b}
  html,body{height:100%;margin:0;font-family:Arial,Helvetica,sans-serif;background:var(--bg);color:var(--fg)}
  #gameWrap{display:flex;flex-direction:column;height:100vh;align-items:center;justify-content:center}
  canvas{background:linear-gradient(#071226,#0b1220);border-radius:12px;box-shadow:0 8px 24px rgba(0,0,0,.6)}
  #ui{width:100%;max-width:900px;margin-top:12px;display:flex;justify-content:space-between;gap:8px;align-items:center}
  .btn{background:var(--accent);color:#111;padding:8px 12px;border-radius:8px;border:none;font-weight:700;cursor:pointer}
  .muted{opacity:.8;font-size:13px}
  #overlay{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;pointer-events:none}
  .panel{pointer-events:auto;background:rgba(10,14,20,.95);color:var(--fg);padding:18px;border-radius:12px;box-shadow:0 12px 50px rgba(0,0,0,.7)}
  .hidden{display:none}
  #shop .item{display:flex;justify-content:space-between;padding:10px;border-bottom:1px solid rgba(255,255,255,.03)}
  small{opacity:.7}
</style>
</head>
<body>
<div id="gameWrap">
  <canvas id="c"></canvas>

  <div id="ui">
    <div class="muted">Puntuación: <span id="score">0</span></div>
    <div style="display:flex;gap:8px">
      <button class="btn" id="startBtn">Jugar</button>
      <button class="btn" id="shopBtn">Tienda</button>
      <button class="btn" id="shareBtn">Compartir</button>
    </div>
  </div>
</div>

<!-- Overlay panels -->
<div id="overlay">
  <div id="menu" class="panel">
    <h2>Canyon Dash</h2>
    <p class="muted">Evita obstáculos, recoge monedas y corre lo más lejos que puedas.</p>
    <p><button class="btn" id="playNow">Comenzar</button></p>
    <small class="muted">Controles: tecla espacio / tap para saltar. Mantén pulsado para salto largo.</small>
  </div>

  <div id="shop" class="panel hidden">
    <h3>Tienda</h3>
    <div id="shopItems" style="min-width:280px">
      <div class="item"><div>Mejora salto (+10%)</div><div><button class="btn buy" data-id="jump1">100</button></div></div>
      <div class="item"><div>Multiplicador x2 (1 partida)</div><div><button class="btn buy" data-id="mult1">250</button></div></div>
      <div style="text-align:right;margin-top:8px"><button class="btn" id="closeShop">Cerrar</button></div>
    </div>
  </div>
</div>

<script>
/* --- Canyon Dash: juego HTML5 simple --- 
 - Canvas, player, obstacles, coins
 - Score, coins, simple shop stored in localStorage
 - Easy to extend: add sprites, audio, levels
*/

(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d', { alpha: false });

  // Responsive sizing
  function fit() {
    const maxW = Math.min(window.innerWidth - 40, 900);
    const maxH = Math.min(window.innerHeight - 140, 640);
    canvas.width = Math.floor(maxW);
    canvas.height = Math.floor(maxH);
  }
  fit();
  window.addEventListener('resize', fit);

  // UI elements
  const overlay = document.getElementById('overlay');
  const menu = document.getElementById('menu');
  const shopPanel = document.getElementById('shop');
  const startBtn = document.getElementById('startBtn');
  const playNow = document.getElementById('playNow');
  const shopBtn = document.getElementById('shopBtn');
  const closeShop = document.getElementById('closeShop');
  const scoreEl = document.getElementById('score');
  const buyBtns = document.getElementsByClassName('buy');
  const shareBtn = document.getElementById('shareBtn');

  // Game state
  let running = false;
  let lastTime=0;
  let speed = 240; // px/sec
  let scroll = 0;
  let distance = 0;
  let score = 0;
  let coins = 0;

  // Player
  const player = {
    x: 80,
    y: 0,
    w: 38,
    h: 48,
    vy: 0,
    grounded: false,
    jumpPower: 520, // base
    gravity: 1600,
    color: '#ffd28a',
    dash: false
  };

  // Obstacles & coins
  let obstacles = [];
  let spawnTimer = 0;
  let coinTimer = 0;

  // Shop / upgrades (saved)
  const storage = JSON.parse(localStorage.getItem('cdash_save')||'{}');
  const upgrades = {
    jump1: { unlocked: false, price: 100, apply: ()=>{player.jumpPower *= 1.10} },
    mult1: { unlocked: false, price: 250, apply: ()=>{ /* one-time multiplier handled at run */ } }
  };
  let oneTimeMul = false;

  // Load saved
  if(storage.coins) coins = storage.coins;
  if(storage.upgrades){
    for(const id in storage.upgrades) if(upgrades[id]) upgrades[id].unlocked = storage.upgrades[id];
  }

  function save() {
    localStorage.setItem('cdash_save', JSON.stringify({
      coins,
      upgrades: Object.fromEntries(Object.entries(upgrades).map(([k,v])=>[k,v.unlocked]))
    }));
  }

  // Input
  let pressing = false;
  function onKey(e){
    if(e.code === 'Space' || e.code === 'ArrowUp') {
      e.preventDefault();
      startJump();
    }
    if(e.code === 'KeyR' && !running){
      startGame();
    }
  }
  window.addEventListener('keydown', onKey);
  window.addEventListener('keyup', ()=>pressing=false);
  window.addEventListener('touchstart', (e)=>{ e.preventDefault(); startJump(); }, {passive:false});

  function startJump(){
    pressing = true;
    if(!running) return startGame();
    if(player.grounded){
      player.vy = -player.jumpPower;
      player.grounded = false;
    } else {
      // allow longer jump if holding
      if(player.vy < 0) player.vy -= 8;
    }
  }

  // Game lifecycle
  function startGame(){
    // reset state
    running = true;
    overlay.style.display = 'none';
    menu.classList.add('hidden');
    shopPanel.classList.add('hidden');
    obstacles = [];
    spawnTimer = 0;
    coinTimer = 0;
    scroll = 0;
    distance = 0;
    score = 0;
    // apply persistent upgrades (reapply)
    player.jumpPower = 520;
    for(const id in upgrades) if(upgrades[id].unlocked) upgrades[id].apply();
    oneTimeMul = false;
    player.y = canvas.height - 140;
    player.vy = 0;
    player.grounded = true;
    lastTime = performance.now();
    requestAnimationFrame(loop);
  }

  function endGame(){
    running = false;
    // reward coins by distance
    const reward = Math.floor(distance / 25);
    coins += reward;
    // if user bought multiplier one-time (example), apply
    if(upgrades.mult1.unlocked && !oneTimeMul){
      coins += reward; // double reward
      oneTimeMul = true;
    }
    save();
    // show menu with score
    menu.innerHTML = `<h2>Fin — Puntuación: ${score}</h2>
      <p class="muted">Distancia: ${Math.floor(distance)} — Monedas obtenidas: ${reward}</p>
      <p><button class="btn" id="playNow2">Jugar otra vez</button>
         <button class="btn" id="toShop">Tienda</button></p>
      <small class="muted">Monedas totales: ${coins}</small>`;
    overlay.style.display='flex';
    menu.classList.remove('hidden');
    // attach listeners for new buttons
    document.getElementById('playNow2').addEventListener('click', startGame);
    document.getElementById('toShop').addEventListener('click', ()=>{ openShop(); });
  }

  // Obstacles spawn
  function spawnObstacle() {
    const h = 24 + Math.random()*60;
    obstacles.push({
      x: canvas.width + 40,
      y: canvas.height - 120 - h,
      w: 28 + Math.random()*36,
      h: h,
      color: '#b05b4a',
      type: 'rock'
    });
  }
  function spawnCoin(){
    obstacles.push({
      x: canvas.width + 40,
      y: canvas.height - 140 - (60 + Math.random()*120),
      w: 18, h: 18, color:'#ffd960', type:'coin'
    });
  }

  // Main loop
  function loop(now){
    const dt = Math.min(0.05, (now - lastTime)/1000);
    lastTime = now;
    update(dt);
    render();
    if(running) requestAnimationFrame(loop);
  }

  function update(dt){
    // speed ramp
    speed += dt * 2; // very slow ramp
    scroll += speed * dt;
    distance += speed * dt;

    // spawn
    spawnTimer -= dt;
    if(spawnTimer <= 0){
      spawnObstacle();
      spawnTimer = 0.8 + Math.random()*1.0 - Math.min(distance/15000,0.6); // become more frequent
    }
    coinTimer -= dt;
    if(coinTimer <= 0){
      if(Math.random() < 0.6) spawnCoin();
      coinTimer = 0.6 + Math.random()*1.4;
    }

    // physics
    player.vy += player.gravity * dt;
    player.y += player.vy * dt;
    // ground collision
    const groundY = canvas.height - 100;
    if(player.y + player.h/2 >= groundY){
      player.y = groundY - player.h/2;
      player.vy = 0;
      player.grounded = true;
    } else player.grounded = false;

    // obstacles movement & collision
    for(let i=obstacles.length-1;i>=0;i--){
      const o = obstacles[i];
      o.x -= speed * dt;
      // collision player
      if(rectColl(player.x - player.w/2, player.y - player.h/2, player.w, player.h, o.x - o.w/2, o.y - o.h/2, o.w, o.h)){
        if(o.type === 'coin'){
          score += 50;
          coins += 1;
          obstacles.splice(i,1);
          save();
        } else {
          // hit obstacle -> game over
          return endGame();
        }
      }
      // off screen
      if(o.x + o.w < -40) obstacles.splice(i,1);
    }

    // score increments by distance
    score = Math.floor(distance);
    scoreEl.textContent = score;
  }

  function rectColl(ax,ay,aw,ah,bx,by,bw,bh){
    return ax < bx+bw && ax+aw > bx && ay < by+bh && ay+ah > by;
  }

  function render(){
    // clear
    ctx.fillStyle = '#071226';
    ctx.fillRect(0,0,canvas.width,canvas.height);

    // parallax background (simple)
    drawMountains();
    // ground
    ctx.fillStyle = '#10304a';
    ctx.fillRect(0, canvas.height - 96, canvas.width, 96);

    // draw obstacles
    for(const o of obstacles){
      if(o.type === 'coin'){
        ctx.fillStyle = o.color;
        ctx.beginPath();
        ctx.arc(o.x, o.y, o.w/2, 0, Math.PI*2);
        ctx.fill();
        ctx.fillStyle='#fff8'; ctx.font='10px Arial'; ctx.fillText('¢', o.x-4, o.y+3);
      } else {
        ctx.fillStyle = o.color;
        roundRect(ctx, o.x - o.w/2, o.y - o.h/2, o.w, o.h, 6, true, false);
      }
    }

    // draw player
    ctx.save();
    ctx.translate(player.x, player.y);
    // shadow
    ctx.fillStyle = 'rgba(0,0,0,.25)';
    ctx.beginPath(); ctx.ellipse(0, player.h/2 + 14, 28, 10, 0, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = player.color;
    roundRect(ctx, -player.w/2, -player.h/2, player.w, player.h, 6, true, false);
    ctx.fillStyle = '#fff8'; ctx.font='12px Arial'; ctx.fillText(':-)', -10, 4);
    ctx.restore();

    // HUD: coins
    ctx.fillStyle = '#ffd960';
    ctx.font = '16px Arial';
    ctx.fillText('Monedas: ' + coins, 12, 24);
  }

  function drawMountains(){
    const baseH = canvas.height - 96;
    ctx.fillStyle = '#0d2b3f';
    for(let i=0;i<3;i++){
      ctx.beginPath();
      const offset = (scroll * (0.2 + i*0.15)) % (canvas.width + 400);
      ctx.moveTo(-400 + offset, baseH);
      ctx.lineTo(150 + offset, baseH - (120 + i*40));
      ctx.lineTo(400 + offset, baseH);
      ctx.closePath();
      ctx.fill();
    }
  }

  function roundRect(ctx,x,y,w,h,r,fill,stroke){
    if(typeof r === 'number') r = {tl:r,tr:r,br:r,bl:r};
    ctx.beginPath();
    ctx.moveTo(x+r.tl,y);
    ctx.lineTo(x+w-r.tr,y);
    ctx.quadraticCurveTo(x+w,y,x+w,y+r.tr);
    ctx.lineTo(x+w,y+h-r.br);
    ctx.quadraticCurveTo(x+w,y+h,x+w-r.br,y+h);
    ctx.lineTo(x+r.bl,y+h);
    ctx.quadraticCurveTo(x,y+h,x,y+h-r.bl);
    ctx.lineTo(x,y+r.tl);
    ctx.quadraticCurveTo(x,y,x+r.tl,y);
    ctx.closePath();
    if(fill) ctx.fill();
    if(stroke) ctx.stroke();
  }

  // UI controls
  startBtn.addEventListener('click', ()=>{ if(!running) startGame(); else endGame(); });
  playNow.addEventListener('click', startGame);
  shopBtn.addEventListener('click', openShop);
  closeShop.addEventListener('click', ()=>{ overlay.style.display='none'; shopPanel.classList.add('hidden'); menu.classList.add('hidden'); });

  function openShop(){
    overlay.style.display='flex';
    menu.classList.add('hidden');
    shopPanel.classList.remove('hidden');
    // update buttons state
    for(const b of buyBtns){
      const id = b.dataset.id;
      const up = upgrades[id];
      b.disabled = up.unlocked || (coins < up.price);
      b.textContent = up.unlocked ? 'Comprado' : up.price;
      b.onclick = ()=>{ buyUpgrade(id); };
    }
  }

  function buyUpgrade(id){
    const up = upgrades[id];
    if(up.unlocked) return;
    if(coins < up.price) { alert('No tienes suficientes monedas.'); return; }
    coins -= up.price;
    up.unlocked = true;
    up.apply();
    save();
    openShop(); // refresh
  }

  // sharing
  shareBtn.addEventListener('click', ()=>{
    if(navigator.share){
      navigator.share({title:'Canyon Dash', text:`Mi puntuación: ${score}`, url:location.href});
    } else {
      prompt('Comparte esta URL:', location.href);
    }
  });

  // initialize menu coin display
  menu.querySelectorAll('small').forEach(s=>s.textContent = `Monedas totales: ${coins}`);

  // initial draw
  render();

  // expose for debugging
  window.cDash = { startGame, endGame, save, upgrades, getCoins: ()=>coins };

})();
</script>
</body>
</html>


---
