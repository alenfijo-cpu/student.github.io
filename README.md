# students.github.io
<!--
File: fighting-game.html
Single-file 2D fighting game (HTML5 Canvas + JavaScript)
Instructions:
  - Save this file as fighting-game.html
  - Open locally in a browser to play, or push to a GitHub repo and enable GitHub Pages
Controls:
  Player 1: A (left), D (right), W (jump), F (attack)
  Player 2: ArrowLeft, ArrowRight, ArrowUp (jump), Numpad0 or '/' (attack)
Features included:
  - Two players, health bars, simple physics (gravity), jump, walk, attack with hitbox,
  - Knockback, round restart, win counting, simple particle effects for hits
  - All graphics drawn with canvas (no external assets)

Want: sprite animations, sound, combo system, AI, networked multiplayer? Tell me and I'll add.
-->

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>2D Fighting Game (HTML5 Canvas)</title>
  <style>
    html,body{height:100%;margin:0;background:#111;color:#eee;font-family:system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial}
    .wrap{display:flex;flex-direction:column;align-items:center;gap:12px;padding:18px}
    canvas{background: linear-gradient(#20232a,#121416); border:6px solid #222; border-radius:8px}
    .ui{display:flex;gap:12px;width:820px;justify-content:space-between}
    .hud{display:flex;flex-direction:column;gap:6px}
    .health{height:18px;width:300px;background:#333;border-radius:6px;overflow:hidden}
    .bar{height:100%;border-radius:6px}
    .controls{font-size:13px;opacity:.9}
    button{padding:8px 12px;border-radius:6px;border:none;background:#2d8cff;color:#fff}
  </style>
</head>
<body>
  <div class="wrap">
    <h1>2D Fighting Game — local 2-player</h1>
    <div class="ui">
      <div class="hud">
        <div>Player 1 <span id="p1wins">Wins: 0</span></div>
        <div class="health"><div id="p1bar" class="bar" style="background:linear-gradient(90deg,#8ef,#08f);width:100%"></div></div>
      </div>
      <div style="align-self:center">Round: <span id="round">1</span> — Press R to reset</div>
      <div class="hud" style="align-items:flex-end">
        <div>Player 2 <span id="p2wins">Wins: 0</span></div>
        <div class="health"><div id="p2bar" class="bar" style="background:linear-gradient(90deg,#f88,#f00);width:100%"></div></div>
      </div>
    </div>

    <canvas id="game" width="800" height="400"></canvas>
    <div style="width:820px;display:flex;justify-content:space-between;align-items:center">
      <div class="controls">
        <strong>Player 1:</strong> A/D = move, W = jump, F = attack
        &nbsp; &nbsp;
        <strong>Player 2:</strong> ←/→ = move, ↑ = jump, / (slash) or Numpad0 = attack
      </div>
      <div>
        <button id="restart">Restart Round</button>
      </div>
    </div>
  </div>

<script>
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

// Game constants
const G = 0.8; // gravity
const FRICTION = 0.9;
const GROUND_Y = H - 60;
const PLAYER_W = 48, PLAYER_H = 72;

let round = 1;
let p1Wins = 0, p2Wins = 0;

const ui = {
  p1bar: document.getElementById('p1bar'),
  p2bar: document.getElementById('p2bar'),
  p1wins: document.getElementById('p1wins'),
  p2wins: document.getElementById('p2wins'),
  round: document.getElementById('round')
}

class Player {
  constructor(x,color,controls){
    this.x = x; this.y = GROUND_Y - PLAYER_H;
    this.vx = 0; this.vy = 0; this.color = color;
    this.facing = 1; // 1 = right, -1 = left
    this.onGround = true;
    this.health = 100;
    this.maxHealth = 100;
    this.attacking = false;
    this.attackTimer = 0;
    this.controls = controls; // object with keys for left,right,jump,attack
    this.score = 0;
    this.combo = 0;
  }
  rect(){return {x:this.x,y:this.y,w:PLAYER_W,h:PLAYER_H}}
  attackHitbox(){
    const reach = 28;
    if(!this.attacking) return null;
    if(this.facing===1) return {x:this.x+PLAYER_W, y:this.y+20, w:reach, h:20};
    return {x:this.x - reach, y:this.y+20, w:reach, h:20};
  }
  update(input){
    // Horizontal movement
    if(input.left) { this.vx -= 1.3; this.facing = -1; }
    if(input.right) { this.vx += 1.3; this.facing = 1; }
    // jump
    if(input.jump && this.onGround){ this.vy = -15; this.onGround = false; }
    // attack
    if(input.attack && !this.attacking){ this.attacking = true; this.attackTimer = 16; }

    // physics
    this.vy += G;
    this.vx *= FRICTION;
    this.x += this.vx;
    this.y += this.vy;

    // bounds
    if(this.y + PLAYER_H >= GROUND_Y){ this.y = GROUND_Y - PLAYER_H; this.vy = 0; this.onGround = true; }
    if(this.x < 10) { this.x = 10; this.vx = 0; }
    if(this.x + PLAYER_W > W-10){ this.x = W-10-PLAYER_W; this.vx = 0; }

    if(this.attacking){
      this.attackTimer -= 1;
      if(this.attackTimer <= 0){ this.attacking = false; this.attackTimer = 0; }
    }
  }
  draw(ctx){
    // simple body
    ctx.save();
    ctx.translate(this.x + PLAYER_W/2, this.y + PLAYER_H/2);
    if(this.facing===-1) ctx.scale(-1,1);
    ctx.translate(-PLAYER_W/2, -PLAYER_H/2);
    // torso
    roundRect(ctx, 6, 6, PLAYER_W-12, PLAYER_H-24, 8, this.color);
    // head
    circle(ctx, PLAYER_W/2, 10, 12, '#222');
    // legs
    roundRect(ctx, 10, PLAYER_H-20, 14, 14, 4, '#111');
    roundRect(ctx, PLAYER_W-24, PLAYER_H-20, 14, 14, 4, '#111');
    // attack slash
    if(this.attacking){
      ctx.globalAlpha = 0.9;
      ctx.fillStyle = 'rgba(255,255,255,0.08)';
      ctx.fillRect(PLAYER_W-6, 20, 28, 22);
    }
    ctx.restore();
  }
}

function roundRect(ctx,x,y,w,h,r,fill){ctx.beginPath();ctx.moveTo(x+r,y);ctx.arcTo(x+w,y,x+w,y+h,r);ctx.arcTo(x+w,y+h,x,y+h,r);ctx.arcTo(x,y+h,x,y,r);ctx.arcTo(x,y,x+w,y,r);ctx.closePath();ctx.fillStyle=fill;ctx.fill();}
function circle(ctx,x,y,r,fill){ctx.beginPath();ctx.arc(x,y,r,0,Math.PI*2);ctx.fillStyle=fill;ctx.fill();}

// players and inputs
const input = {
  p1: {left:false,right:false,jump:false,attack:false},
  p2: {left:false,right:false,jump:false,attack:false}
}

const p1 = new Player(120,'#0aa', {left:'a', right:'d', jump:'w', attack:'f'});
const p2 = new Player(W-180,'#a22', {left:'ArrowLeft', right:'ArrowRight', jump:'ArrowUp', attack:'/'});

// keyboard handling
const keysDown = {};
window.addEventListener('keydown', e=>{
  if(e.key==='r' || e.key==='R') resetRound();
  keysDown[e.key] = true;
  // support numpad 0 as attack also
  if(e.code === 'Numpad0') keysDown['/'] = true; // alias
});
window.addEventListener('keyup', e=>{ delete keysDown[e.key]; if(e.code==='Numpad0') delete keysDown['/']; });

function pollInputs(){
  input.p1.left = !!(keysDown['a'] || keysDown['A']);
  input.p1.right = !!(keysDown['d'] || keysDown['D']);
  input.p1.jump = !!(keysDown['w'] || keysDown['W']);
  input.p1.attack = !!(keysDown['f'] || keysDown['F']);

  input.p2.left = !!(keysDown['ArrowLeft']);
  input.p2.right = !!(keysDown['ArrowRight']);
  input.p2.jump = !!(keysDown['ArrowUp']);
  input.p2.attack = !!(keysDown['/']);
}

// simple particle system for hits
const particles = [];
function spawnHit(x,y,color){
  for(let i=0;i<10;i++) particles.push({x,y,vx:(Math.random()*6-3), vy:(Math.random()*-4-1), life:30, color});
}

function updateParticles(){
  for(let i=particles.length-1;i>=0;i--){
    const p = particles[i]; p.x+=p.vx; p.y+=p.vy; p.vy+=0.2; p.life--; if(p.life<=0) particles.splice(i,1);
  }
}

// collision helpers
function rectsOverlap(a,b){return a.x < b.x+b.w && a.x+a.w > b.x && a.y < b.y+b.h && a.y+a.h > b.y}

let roundActive = true;
let roundTimer = 0;

function gameLoop(){
  pollInputs();
  if(roundActive){
    p1.update(input.p1);
    p2.update(input.p2);

    // facing logic
    if(p1.x < p2.x) { p1.facing = 1; p2.facing = -1; }
    else { p1.facing = -1; p2.facing = 1; }

    // attacks
    [ [p1,p2], [p2,p1] ].forEach(([att,def])=>{
      const h = att.attackHitbox();
      if(h){
        const defRect = def.rect();
        if(rectsOverlap(h,defRect)){
          // apply damage once per attack activation
          if(att.attackTimer === 8){
            def.health -= 12; att.combo++; def.vx += (att.facing*6); def.vy = -4;
            spawnHit(def.x + PLAYER_W/2, def.y + 30, att.color);
          }
        }
      }
    });

    // clamp health
    p1.health = Math.max(0,p1.health);
    p2.health = Math.max(0,p2.health);

    // round end
    if(p1.health <= 0 || p2.health <= 0){
      roundActive = false;
      roundTimer = 90;
      if(p1.health <= 0) p2Wins++;
      if(p2.health <= 0) p1Wins++;
      ui.p1wins.textContent = 'Wins: ' + p1Wins;
      ui.p2wins.textContent = 'Wins: ' + p2Wins;
    }
  } else {
    roundTimer -= 1;
    if(roundTimer <= 0) newRound();
  }

  updateParticles();
  draw();
  requestAnimationFrame(gameLoop);
}

function draw(){
  ctx.clearRect(0,0,W,H);
  // ground
  ctx.fillStyle = '#0b0b0b'; ctx.fillRect(0,GROUND_Y,H, H);
  ctx.fillStyle = '#222'; ctx.fillRect(0,GROUND_Y, W, 6);

  // background details
  for(let i=0;i<6;i++){ ctx.fillStyle = 'rgba(255,255,255,0.02)'; ctx.fillRect(60 + i*120, 40, 100, 180); }

  // draw players
  p1.draw(ctx); p2.draw(ctx);

  // draw health bars
  ui.p1bar.style.width = (p1.health / p1.maxHealth * 100) + '%';
  ui.p2bar.style.width = (p2.health / p2.maxHealth * 100) + '%';

  // draw attack hitboxes (debug)
  [p1,p2].forEach(p=>{
    const h = p.attackHitbox();
    if(h){ ctx.fillStyle='rgba(255,255,255,0.08)'; ctx.fillRect(h.x,h.y,h.w,h.h); }
  });

  // draw particle effects
  for(const pp of particles){ ctx.globalAlpha = pp.life/30; circle(ctx, pp.x, pp.y, 3, pp.color); ctx.globalAlpha=1; }

  // draw names / HP numbers
  ctx.fillStyle = '#fff'; ctx.font = '14px monospace'; ctx.fillText('P1 ' + Math.round(p1.health), 16, 22);
  ctx.fillText('P2 ' + Math.round(p2.health), W-70, 22);

  if(!roundActive){
    ctx.fillStyle = 'rgba(0,0,0,0.6)'; ctx.fillRect(0,0,W,H);
    ctx.fillStyle = '#fff'; ctx.font = '32px sans-serif'; ctx.textAlign='center';
    const winner = p1.health<=0 ? 'Player 2 Wins!' : 'Player 1 Wins!';
    ctx.fillText(winner, W/2, H/2 - 10);
    ctx.font = '16px sans-serif'; ctx.fillText('New round starts automatically...', W/2, H/2 + 20);
    ctx.textAlign='start';
  }
}

function resetRound(){
  p1.x = 120; p2.x = W-180; p1.health = p1.maxHealth; p2.health = p2.maxHealth; p1.vx = p1.vy = p2.vx = p2.vy = 0; roundActive = true; particles.length = 0; ui.round.textContent = round; }

function newRound(){ round++; ui.round.textContent = round; resetRound(); }

document.getElementById('restart').addEventListener('click', ()=>{ resetRound(); });

// start
resetRound();
requestAnimationFrame(gameLoop);
</script>
</body>
</html>
