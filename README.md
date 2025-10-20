<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Glitch Shift — Prototype</title>
<style>
  html,body{height:100%;margin:0;background:#0b0f1a;color:#dfe8ff;font-family:Inter,ui-sans-serif,system-ui,Segoe UI,Roboto,Helvetica,Arial;}
  #game{display:block;margin:0 auto; background:#081028; box-shadow:0 10px 30px rgba(2,6,23,.7);}
  .ui{width:800px;margin:12px auto;text-align:center}
  .hint{opacity:.9;font-size:14px}
  button{background:#182036;border:none;padding:8px 12px;border-radius:8px;color:#cfe1ff;cursor:pointer}
</style>
</head>
<body>
<div class="ui">
  <h2>Glitch Shift — Playable Prototype</h2>
  <div class="hint">Move: ← → or A D • Jump: ↑ / W / Space • Shift Mode: F • Restart: R</div>
  <div style="margin-top:8px"><button id="restart">Restart</button></div>
</div>
<canvas id="game" width="800" height="500" tabindex="0"></canvas>
<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  let width = canvas.width; let height = canvas.height;

  // Game state
  let keys = {};
  let mode = 'stable'; // 'stable' or 'glitch'
  let fragmentsCollected = 0;
  let totalFragments = 5;
  let gameOver = false;

  function reset(){
    player.x = 60; player.y = 380; player.vx = 0; player.vy = 0; fragmentsCollected = 0; mode='stable'; gameOver=false; initEntities();
  }

  // Player
  const player = {x:60,y:380,w:28,h:38,vx:0,vy:0,onGround:false};

  // Level data
  let platforms = [];
  let fragments = [];

  function initEntities(){
    platforms = [
      {x:0,y:460,w:800,h:40},
      {x:120,y:380,w:120,h:16},
      {x:300,y:320,w:160,h:16},
      {x:520,y:260,w:120,h:16},
      {x:680,y:200,w:80,h:16},
      {x:420,y:430,w:80,h:16}
    ];
    fragments = [
      {x:160,y:342, r:8, id:1},
      {x:360,y:282, r:8, id:2},
      {x:560,y:222, r:8, id:3},
      {x:720,y:162, r:8, id:4},
      {x:460,y:392, r:8, id:5}
    ];
  }

  initEntities();

  // physics tuning
  const GRAV = 0.85;
  const FRICTION = 0.85;

  function update(){
    if(gameOver) return;
    // input
    if(keys['ArrowLeft']||keys['a']||keys['A']) player.vx -= 0.9;
    if(keys['ArrowRight']||keys['d']||keys['D']) player.vx += 0.9;
    if((keys['ArrowUp']||keys['w']||keys['W']||keys[' ']) && player.onGround){ player.vy = -15*(mode==='stable'?1: -1); player.onGround=false; }

    // apply gravity (mode affects direction)
    const gravity = (mode==='stable')? GRAV : -GRAV;
    player.vy += gravity;

    // clamp
    player.vx *= 0.92;
    if(Math.abs(player.vx)<0.05) player.vx=0;

    player.x += player.vx;
    player.y += player.vy;

    player.onGround = false;
    // collision with platforms
    for(const p of platforms){
      if(player.x + player.w > p.x && player.x < p.x + p.w){
        if(mode==='stable'){
          // normal: check downward collision
          if(player.y + player.h > p.y && player.y + player.h < p.y + Math.abs(player.vy) + 22 && player.vy >=0){
            player.y = p.y - player.h; player.vy = 0; player.onGround = true;
          }
        } else {
          // glitch mode: gravity reversed, platforms act as ceiling
          if(player.y < p.y + p.h && player.y > p.y - Math.abs(player.vy) - 22 && player.vy <=0){
            player.y = p.y + p.h; player.vy = 0; player.onGround = true;
          }
        }
      }
    }

    // bounds
    if(player.x < 0) player.x = 0, player.vx = 0;
    if(player.x + player.w > width) player.x = width - player.w, player.vx = 0;
    if(player.y > height+50 || player.y < -200) {
      // falling off
      gameOver = true;
    }

    // collect fragments (fragments only collectible in matching mode sometimes)
    for(let i=fragments.length-1;i>=0;i--){
      const f = fragments[i];
      const dx = (player.x + player.w/2) - f.x;
      const dy = (player.y + player.h/2) - f.y;
      if(Math.hypot(dx,dy) < f.r + Math.max(player.w,player.h)/2){
        fragments.splice(i,1); fragmentsCollected++;
        // small reward: boost
        player.vy -= (mode==='stable')?  -6 : 6;
      }
    }

    if(fragmentsCollected >= totalFragments){
      gameOver = true; // victory
    }
  }

  function draw(){
    // background
    if(mode==='stable'){
      ctx.fillStyle = '#071127'; ctx.fillRect(0,0,width,height);
      // subtle grid
      ctx.strokeStyle = 'rgba(255,255,255,0.03)'; ctx.lineWidth=1;
      for(let x=0;x<width;x+=40){ctx.beginPath();ctx.moveTo(x,0);ctx.lineTo(x,height);ctx.stroke();}
    } else {
      // glitched background
      ctx.fillStyle = '#0d0016'; ctx.fillRect(0,0,width,height);
      // scanlines
      ctx.fillStyle = 'rgba(255,255,255,0.02)';
      for(let y=0;y<height;y+=6){ctx.fillRect(0,y,width,2);}    
    }

    // platforms
    for(const p of platforms){
      const grad = ctx.createLinearGradient(p.x,p.y,p.x,p.y+p.h);
      if(mode==='stable') { grad.addColorStop(0,'#2c3a58'); grad.addColorStop(1,'#152138'); }
      else { grad.addColorStop(0,'#3b003b'); grad.addColorStop(1,'#120018'); }
      ctx.fillStyle = grad; ctx.fillRect(p.x,p.y,p.w,p.h);
      // outline
      ctx.strokeStyle = 'rgba(255,255,255,0.06)'; ctx.strokeRect(p.x,p.y,p.w,p.h);
    }

    // fragments
    for(const f of fragments){
      ctx.beginPath(); ctx.arc(f.x,f.y,f.r,0,Math.PI*2);
      if(mode==='stable') ctx.fillStyle='#ffe27a'; else ctx.fillStyle='#ff66ff';
      ctx.fill();
      // sparkle
      ctx.strokeStyle='rgba(255,255,255,0.25)'; ctx.stroke();
    }

    // player
    ctx.save();
    if(mode==='stable'){
      // glowing outline
      ctx.fillStyle = '#9ad3ff'; ctx.fillRect(player.x,player.y,player.w,player.h);
    } else {
      // skewed & neon
      ctx.fillStyle = '#ff9aff'; ctx.fillRect(player.x,player.y,player.w,player.h);
      // ghost afterimage
      ctx.globalAlpha = 0.35;
      ctx.fillRect(player.x-8,player.y+6,player.w,player.h);
      ctx.globalAlpha = 1;
    }
    ctx.restore();

    // HUD
    ctx.fillStyle = 'rgba(255,255,255,0.9)'; ctx.font='16px ui-sans-serif';
    ctx.fillText(`Mode: ${mode.toUpperCase()}   Fragments: ${fragmentsCollected}/${totalFragments}` , 12, 24);

    if(gameOver){
      ctx.fillStyle = 'rgba(0,0,0,0.5)'; ctx.fillRect(0,0,width,height);
      ctx.fillStyle = '#fff'; ctx.textAlign='center'; ctx.font='28px ui-sans-serif';
      if(fragmentsCollected>=totalFragments) ctx.fillText('CORE REBOOTED — YOU WIN! (R to restart)', width/2, height/2);
      else ctx.fillText('YOU GOT ERASED — R to restart', width/2, height/2);
      ctx.textAlign='left';
    }
  }

  // game loop
  let last = performance.now();
  function loop(now){
    const dt = now - last; last = now;
    update(); draw();
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);

  // input
  window.addEventListener('keydown', e => { keys[e.key]=true; if(e.key==='f' || e.key==='F'){ mode = (mode==='stable')? 'glitch' : 'stable'; /* small visual jitter */ } if(e.key==='r' || e.key==='R'){ reset(); } });
  window.addEventListener('keyup', e => { keys[e.key]=false; });
  canvas.addEventListener('click', ()=> canvas.focus());

  // touch-friendly simple controls (left/right/jump)
  // tiny on-screen controls for mobile
  function makeBtn(txt, x, y, w, h, onDown, onUp){
    const b = document.createElement('div');
    Object.assign(b.style,{position:'fixed',left:x+'px',top:y+'px',width:w+'px',height:h+'px',opacity:.25,backdropFilter:'blur(2px)'});
    b.innerText=txt; b.style.display='flex'; b.style.alignItems='center'; b.style.justifyContent='center'; b.style.borderRadius='10px'; b.style.fontSize='18px'; b.style.color='#fff';
    document.body.appendChild(b);
    b.addEventListener('touchstart', (ev)=>{ev.preventDefault(); onDown();}); b.addEventListener('touchend', (ev)=>{ev.preventDefault(); onUp();});
  }
  // small buttons
  makeBtn('◀', 8, height+70, 56,56, ()=>keys['ArrowLeft']=true, ()=>keys['ArrowLeft']=false);
  makeBtn('▶', 72, height+70, 56,56, ()=>keys['ArrowRight']=true, ()=>keys['ArrowRight']=false);
  makeBtn('▲', 136, height+70, 56,56, ()=>{keys['ArrowUp']=true; setTimeout(()=>keys['ArrowUp']=false,120);}, ()=>{});
  makeBtn('F', width-120, height+70, 56,56, ()=>{ mode = (mode==='stable')? 'glitch' : 'stable'; }, ()=>{});

  // restart UI
  document.getElementById('restart').addEventListener('click', reset);

  // focus canvas to capture keys
  canvas.focus();
})();
</script>
</body>
</html>
