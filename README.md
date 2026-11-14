# Sky-jumper-extreme
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Sky Jumper Extreme</title>
<style>
* { box-sizing: border-box; }
body {
  margin: 0;
  padding: 15px 10px;
  background: #090e20;
  color: white;
  font-family: Arial, sans-serif;
  text-align: center;
  user-select: none;
}
h1 {
  color: #79f5ff;
  text-shadow: 0 0 18px #258fff;
}
.game {
  width: min(96vw, 420px);
  margin: 15px auto;
  position: relative;
}
canvas {
  display: block;
  width: 100%;
  height: auto;
  border: 3px solid #4de4ff;
  border-radius: 17px;
  touch-action: none;
  background: #182b55;
}
.hud {
  position: absolute;
  top: 12px;
  left: 12px;
  right: 12px;
  display: flex;
  justify-content: space-between;
  pointer-events: none;
}
.hud span {
  background: #101a35aa;
  padding: 8px;
  border-radius: 10px;
}
.overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  background: #071126bb;
}
.panel {
  width: 100%;
  background: linear-gradient(145deg,#233e71,#151d40);
  border: 2px solid #69dcff;
  border-radius: 20px;
  padding: 20px 14px;
}
.panel h2 { color: #80edff; }
button {
  border: 0;
  border-radius: 13px;
  padding: 13px 17px;
  margin: 5px;
  font-size: 17px;
  font-weight: bold;
  cursor: pointer;
  touch-action: none;
  background: #76edff;
}
.play {
  background: linear-gradient(90deg,#73f7c0,#72ddff);
  font-size: 21px;
}
.secondary { background: #d0a5ff; }
.controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: min(96vw,420px);
  margin: 10px auto;
}
.controls button {
  width: 75px;
  height: 65px;
  font-size: 27px;
  background: #243e68;
  color: white;
  border: 2px solid #6adfff;
}
.controls .boost {
  width: 90px;
  font-size: 15px;
  background: #ffca61;
  color: #30200b;
}
.hidden { display: none !important; }
.help { color: #9eafd1; font-size: 12px; }
</style>
</head>
<body>

<h1>🌌 SKY JUMPER</h1>
<p>EXTREME ARCADE ADVENTURE</p>

<div class="game">
<canvas id="game" width="420" height="600"></canvas>

<div class="hud hidden" id="hud">
<span id="score">⭐ 0</span>
<span id="coins">🪙 0</span>
<span id="best">🏆 0</span>
</div>

<div class="overlay" id="menu">
<div class="panel">
<div style="font-size:55px">🚀</div>
<h2>SKY JUMPER</h2>
<p>Jump higher. Collect coins. Avoid monsters!</p>
<p>Best score: <b id="menuBest">0</b></p>
<button class="play" onclick="startGame()">▶ PLAY NOW</button>
<p class="help">Your hero jumps automatically!</p>
</div>
</div>

<div class="overlay hidden" id="pause">
<div class="panel">
<h2>⏸ PAUSED</h2>
<button class="play" onclick="resumeGame()">▶ RESUME</button>
<button class="secondary" onclick="goMenu()">🏠 MENU</button>
</div>
</div>

<div class="overlay hidden" id="over">
<div class="panel">
<h2>💫 GAME OVER</h2>
<p>Your score: <b id="finalScore">0</b></p>
<p>Coins: <b id="finalCoins">0</b></p>
<p>Best: <b id="finalBest">0</b></p>
<button class="play" onclick="startGame()">🔄 PLAY AGAIN</button>
<button class="secondary" onclick="goMenu()">🏠 MENU</button>
</div>
</div>
</div>

<div class="controls">
<div>
<button id="left">⬅️</button>
<button id="right">➡️</button>
</div>
<button class="boost" id="boost">🚀 BOOST</button>
</div>

<p class="help">Touch the game screen or hold arrows to move.</p>
<p class="help">Keyboard: A / D or arrows. Space = boost. P = pause.</p>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
const W = 420, H = 600;

let player, platforms, coins, gems, enemies, particles;
let camera = 0, score = 0, coinCount = 0;
let best = Number(localStorage.getItem("skyJumperBest")) || 0;
let running = false, paused = false;
let lastTime = 0, animation = 0;
let keys = {left:false,right:false};
let audio = null;

document.getElementById("menuBest").textContent = best;

function sound(freq=500,duration=.1) {
 try {
  if(!audio) audio = new AudioContext();
  const osc = audio.createOscillator();
  const gain = audio.createGain();
  osc.frequency.value = freq;
  gain.gain.value = .05;
  osc.connect(gain);
  gain.connect(audio.destination);
  osc.start();
  osc.stop(audio.currentTime + duration);
 } catch(e) {}
}

function random(min,max) {
 return Math.random()*(max-min)+min;
}

function makePlatform(y,x,width=85,type="normal") {
 return {x,y,w:width,h:15,type};
}

function startGame() {
 score=0;
 coinCount=0;
 camera=0;
 particles=[];
 keys.left=false;
 keys.right=false;

 player={
  x:190,y:495,w:34,h:39,
  vx:0,vy:-8.5,
  boostUsed:false,
  invincible:0
 };

 platforms=[
  makePlatform(550,130,160),
  makePlatform(455,270,90),
  makePlatform(360,90,95),
  makePlatform(265,230,100),
  makePlatform(170,65,100),
  makePlatform(75,250,95)
 ];

 coins=[];
 gems=[];
 enemies=[];

 platforms.forEach((p,i)=>{
  if(i>0) addCollectible(p);
 });

 running=true;
 paused=false;

 document.getElementById("menu").classList.add("hidden");
 document.getElementById("pause").classList.add("hidden");
 document.getElementById("over").classList.add("hidden");
 document.getElementById("hud").classList.remove("hidden");

 updateHUD();
 sound(600,.16);

 lastTime=performance.now();
 cancelAnimationFrame(animation);
 animation=requestAnimationFrame(loop);
}

function addCollectible(p) {
 if(Math.random()<.78) {
  coins.push({
   x:p.x+p.w/2,y:p.y-28,
   phase:random(0,6.28),taken:false
  });
 }

 if(Math.random()<.12) {
  gems.push({
   x:p.x+p.w/2,y:p.y-48,
   phase:random(0,6.28),taken:false
  });
 }

 if(Math.random()<.13 && p.y< -100) {
  enemies.push({
   x:random(30,350),y:p.y-40,
   vx:random(1,2)*(Math.random()<.5?1:-1),
   alive:true
  });
 }
}

function addNewPlatforms() {
 let top=platforms.reduce((a,b)=>a.y<b.y?a:b);

 while(top.y>camera-750) {
  const y=top.y-random(78,105);
  const x=random(10,320);
  const type=Math.random()<.12?"moving":
    Math.random()<.13?"spring":"normal";

  top=makePlatform(y,x,random(75,105),type);

  if(type==="moving") top.vx=random(1,2.3);

  platforms.push(top);
  addCollectible(top);
 }

 platforms=platforms.filter(p=>p.y-camera<H+80);
 coins=coins.filter(c=>!c.taken && c.y-camera<H+80);
 gems=gems.filter(g=>!g.taken && g.y-camera<H+80);
 enemies=enemies.filter(e=>e.alive && e.y-camera<H+100);
}

function burst(x,y,color) {
 for(let i=0;i<14;i++) {
  particles.push({
   x,y,vx:random(-3,3),vy:random(-4,2),
   life:random(15,30),color
  });
 }
}

function update(dt) {
 const step=Math.min(dt/16.67,2);

 if(keys.left) player.vx-=.8*step;
 if(keys.right) player.vx+=.8*step;

 player.vx*=Math.pow(.90,step);
 player.vx=Math.max(-7.5,Math.min(7.5,player.vx));

 const oldBottom=player.y+player.h;

 player.x+=player.vx*step;
 player.y+=player.vy*step;
 player.vy+=.34*step;

 if(player.x>W) player.x=-player.w;
 if(player.x+player.w<0) player.x=W;

 platforms.forEach(p=>{
  if(p.type==="moving") {
   p.x+=p.vx*step;
   if(p.x<0 || p.x+p.w>W) p.vx*=-1;
  }
 });

 if(player.vy>0) {
  for(const p of platforms) {
   const newBottom=player.y+player.h;

   if(oldBottom<=p.y+5 && newBottom>=p.y &&
      player.x+player.w>p.x && player.x<p.x+p.w) {

    player.y=p.y-player.h;

    if(p.type==="spring") {
     player.vy=-15;
     burst(player.x+17,p.y,"#ffcf56");
     sound(850,.18);
    } else {
     player.vy=-9.2;
     sound(330,.055);
    }

    player.boostUsed=false;
    break;
   }
  }
 }

 if(player.y-camera<230) camera=player.y-230;

 const height=Math.max(0,Math.floor(-camera));
 score=Math.max(score,height);

 for(const c of coins) {
  if(!c.taken &&
   Math.abs(player.x+17-c.x)<24 &&
   Math.abs(player.y+18-c.y)<27) {

   c.taken=true;
   coinCount++;
   score+=25;
   burst(c.x,c.y,"#ffd84d");
   sound(720,.1);
  }
 }

 for(const g of gems) {
  if(!g.taken &&
   Math.abs(player.x+17-g.x)<25 &&
   Math.abs(player.y+18-g.y)<28) {

   g.taken=true;
   coinCount+=5;
   score+=100;
   burst(g.x,g.y,"#bd7bff");
   sound(1000,.2);
  }
 }

 enemies.forEach(e=>{
  e.x+=e.vx*step;

  if(e.x<5 || e.x>375) e.vx*=-1;

  if(e.alive &&
   player.x+player.w>e.x && player.x<e.x+32 &&
   player.y+player.h>e.y && player.y<e.y+30) {

   if(player.vy>0 && player.y+player.h<e.y+19) {
    e.alive=false;
    player.vy=-11;
    burst(e.x+15,e.y+15,"#91ffb2");
    sound(600,.13);
    score+=50;
   } else if(player.invincible<=0) {
    endGame();
    return;
   }
  }
 });

 if(player.invincible>0) player.invincible-=step;

 if(player.y-camera>H+50) {
  endGame();
  return;
 }

 addNewPlatforms();

 particles.forEach(p=>{
  p.x+=p.vx*step;
  p.y+=p.vy*step;
  p.vy+=.12*step;
  p.life-=step;
 });

 particles=particles.filter(p=>p.life>0);

 updateHUD();
}

function updateHUD() {
 document.getElementById("score").textContent="⭐ "+score;
 document.getElementById("coins").textContent="🪙 "+coinCount;
 document.getElementById("best").textContent="🏆 "+best;
}

function roundRect(x,y,w,h,r,color) {
 ctx.fillStyle=color;
 ctx.beginPath();
 ctx.roundRect(x,y,w,h,r);
 ctx.fill();
}

function drawBackground() {
 const gradient=ctx.createLinearGradient(0,0,0,H);
 gradient.addColorStop(0,"#172d62");
 gradient.addColorStop(.55,"#397eb1");
 gradient.addColorStop(1,"#9bd9e9");

 ctx.fillStyle=gradient;
 ctx.fillRect(0,0,W,H);

 for(let i=0;i<65;i++) {
  const x=(i*79+30)%W;
  const y=((i*137-camera*.25)%(H+50)+H+50)%(H+50);

  ctx.globalAlpha=.25+((i%4)*.15);
  ctx.fillStyle="white";
  ctx.beginPath();
  ctx.arc(x,y,(i%3)+.6,0,Math.PI*2);
  ctx.fill();
 }

 ctx.globalAlpha=1;

 for(let i=0;i<7;i++) {
  const x=(i*83+((i%2)*30))%W;
  const y=((i*119-camera*.17)%(H+100)+H+100)%(H+100);

  ctx.fillStyle="#ffffff25";
  ctx.beginPath();
  ctx.arc(x,y,20,0,Math.PI*2);
  ctx.arc(x+20,y-5,25,0,Math.PI*2);
  ctx.arc(x+43,y,19,0,Math.PI*2);
  ctx.fill();
 }
}

function drawPlatforms() {
 platforms.forEach(p=>{
  const y=p.y-camera;
  if(y<-30 || y>H+30) return;

  let color="#52e59a";
  if(p.type==="moving") color="#55caff";
  if(p.type==="spring") color="#ffcb59";

  roundRect(p.x,y,p.w,p.h,8,color);
  roundRect(p.x,y,p.w,5,4,"#ffffff88");

  if(p.type==="spring") {
   ctx.fillStyle="#68431b";
   ctx.font="16px Arial";
   ctx.textAlign="center";
   ctx.fillText("↟",p.x+p.w/2,y-2);
  }

  if(p.type==="moving") {
   ctx.fillStyle="#123b65";
   ctx.font="13px Arial";
   ctx.textAlign="center";
   ctx.fillText("↔",p.x+p.w/2,y+12);
  }
 });
}

function drawCollectibles() {
 coins.forEach(c=>{
  if(c.taken)return;

  const y=c.y-camera+Math.sin(animation*.005+c.phase)*4;

  ctx.fillStyle="#ffce36";
  ctx.beginPath();
  ctx.arc(c.x,y,10,0,Math.PI*2);
  ctx.fill();

  ctx.strokeStyle="#fff2a4";
  ctx.lineWidth=3;
  ctx.stroke();

  ctx.fillStyle="#87500a";
  ctx.font="bold 11px Arial";
  ctx.textAlign="center";
  ctx.fillText("$",c.x,y+4);
 });

 gems.forEach(g=>{
  if(g.taken)return;

  const y=g.y-camera+Math.sin(animation*.006+g.phase)*5;

  ctx.save();
  ctx.translate(g.x,y);
  ctx.rotate(animation*.002);
  ctx.fillStyle="#c77bff";
  ctx.shadowColor="#d69aff";
  ctx.shadowBlur=15;

  ctx.beginPath();
  ctx.moveTo(0,-13);
  ctx.lineTo(12,0);
  ctx.lineTo(0,13);
  ctx.lineTo(-12,0);
  ctx.closePath();
  ctx.fill();
  ctx.restore();
  ctx.shadowBlur=0;
 });
}

function drawEnemies() {
 enemies.forEach(e=>{
  if(!e.alive)return;

  const y=e.y-camera;

  ctx.fillStyle="#a56bff";
  ctx.beginPath();
  ctx.arc(e.x+16,y+16,16,Math.PI,0);
  ctx.lineTo(e.x+32,y+26);
  ctx.lineTo(e.x,y+26);
  ctx.closePath();
  ctx.fill();

  ctx.fillStyle="white";
  ctx.beginPath();
  ctx.arc(e.x+10,y+14,4,0,Math.PI*2);
  ctx.arc(e.x+22,y+14,4,0,Math.PI*2);
  ctx.fill();

  ctx.fillStyle="#222";
  ctx.beginPath();
  ctx.arc(e.x+10,y+14,1.7,0,Math.PI*2);
  ctx.arc(e.x+22,y+14,1.7,0,Math.PI*2);
  ctx.fill();
 });
}

function drawPlayer() {
 if(player.invincible>0 && Math.floor(animation/80)%2===0)return;

 const x=player.x;
 const y=player.y-camera;
 const bob=Math.sin(animation*.012)*2;

 ctx.fillStyle="#10284b44";
 ctx.beginPath();
 ctx.ellipse(x+17,y+42,15,4,0,0,Math.PI*2);
 ctx.fill();

 roundRect(x-3,y+13+bob,10,18,5,"#ffbd52");
 roundRect(x+3,y+13+bob,28,25,10,"#ff754f");
 roundRect(x+7,y+14+bob,19,8,4,"#ffad7a");

 ctx.fillStyle="#ffd3a4";
 ctx.beginPath();
 ctx.arc(x+17,y+10+bob,14,0,Math.PI*2);
 ctx.fill();

 ctx.fillStyle="#47305b";
 ctx.beginPath();
 ctx.arc(x+17,y+5+bob,13,Math.PI,Math.PI*2);
 ctx.fill();

 ctx.fillStyle="#20253b";
 ctx.beginPath();
 ctx.arc(x+12,y+11+bob,2,0,Math.PI*2);
 ctx.arc(x+22,y+11+bob,2,0,Math.PI*2);
 ctx.fill();

 ctx.strokeStyle="#9a4d4d";
 ctx.lineWidth=1.5;
 ctx.beginPath();
 ctx.arc(x+17,y+13+bob,5,.2,Math.PI-.2);
 ctx.stroke();

 roundRect(x+3,y+34+bob,12,6,3,"#344b91");
 roundRect(x+19,y+34+bob,12,6,3,"#344b91");
}

function drawParticles() {
 particles.forEach(p=>{
  ctx.globalAlpha=Math.max(0,p.life/30);
  ctx.fillStyle=p.color;
  ctx.beginPath();
  ctx.arc(p.x,p.y-camera,3,0,Math.PI*2);
  ctx.fill();
 });

 ctx.globalAlpha=1;
}

function draw() {
 drawBackground();
 drawPlatforms();
 drawCollectibles();
 drawEnemies();
 drawPlayer();
 drawParticles();
}

function loop(time) {
 if(!running || paused)return;

 const dt=Math.min(time-lastTime,34);
 lastTime=time;
 animation=time;

 update(dt);
 draw();

 if(running && !paused) {
  animation=requestAnimationFrame(loop);
 }
}

function endGame() {
 running=false;
 cancelAnimationFrame(animation);
 sound(160,.3);

 if(score>best) {
  best=score;
  try {
   localStorage.setItem("skyJumperBest",best);
  } catch(e) {}
 }

 updateHUD();

 document.getElementById("finalScore").textContent=score;
 document.getElementById("finalCoins").textContent=coinCount;
 document.getElementById("finalBest").textContent=best;
 document.getElementById("over").classList.remove("hidden");
}

function resumeGame() {
 if(!running)return;

 paused=false;
 document.getElementById("pause").classList.add("hidden");

 lastTime=performance.now();
 animation=requestAnimationFrame(loop);
}

function pauseGame() {
 if(!running || paused)return;

 paused=true;
 cancelAnimationFrame(animation);
 document.getElementById("pause").classList.remove("hidden");
}

function goMenu() {
 running=false;
 paused=false;
 cancelAnimationFrame(animation);

 document.getElementById("pause").classList.add("hidden");
 document.getElementById("over").classList.add("hidden");
 document.getElementById("hud").classList.add("hidden");
 document.getElementById("menu").classList.remove("hidden");
 document.getElementById("menuBest").textContent=best;
}

function boost() {
 if(!running || paused || player.boostUsed)return;

 player.vy=-13;
 player.boostUsed=true;
 player.invincible=12;

 burst(player.x+17,player.y+35,"#70edff");
 sound(950,.15);
}

function bindHold(id,direction) {
 const btn=document.getElementById(id);

 btn.addEventListener("pointerdown",e=>{
  e.preventDefault();
  keys[direction]=true;
  btn.setPointerCapture(e.pointerId);
 });

 const release=()=>{
  keys[direction]=false;
 };

 btn.addEventListener("pointerup",release);
 btn.addEventListener("pointercancel",release);
 btn.addEventListener("lostpointercapture",release);
}

bindHold("left","left");
bindHold("right","right");

let canvasPointer=null;

function steerFromCanvas(e) {
 const rect=canvas.getBoundingClientRect();
 const x=(e.clientX-rect.left)*(W/rect.width);
 const center=player.x+player.w/2;

 keys.left=x<center-8;
 keys.right=x>center+8;
}

canvas.addEventListener("pointerdown",e=>{
 if(!running || paused)return;

 e.preventDefault();
 canvasPointer=e.pointerId;
 canvas.setPointerCapture(e.pointerId);
 steerFromCanvas(e);
});

canvas.addEventListener("pointermove",e=>{
 if(canvasPointer===e.pointerId && running && !paused) {
  steerFromCanvas(e);
 }
});

function stopCanvasSteering(e) {
 if(canvasPointer!==null &&
   (!e || e.pointerId===canvasPointer)) {
  canvasPointer=null;
  keys.left=false;
  keys.right=false;
 }
}

canvas.addEventListener("pointerup",stopCanvasSteering);
canvas.addEventListener("pointercancel",stopCanvasSteering);
canvas.addEventListener("lostpointercapture",stopCanvasSteering);

document.getElementById("boost").addEventListener("pointerdown",e=>{
 e.preventDefault();
 boost();
});

document.addEventListener("keydown",e=>{
 if(["ArrowLeft","ArrowRight"," "].includes(e.key)) {
  e.preventDefault();
 }

 if(e.key==="ArrowLeft" || e.key.toLowerCase()==="a") {
  keys.left=true;
 }

 if(e.key==="ArrowRight" || e.key.toLowerCase()==="d") {
  keys.right=true;
 }

 if(e.key===" " && !e.repeat) boost();
 if(e.key.toLowerCase()==="p") pauseGame();
});

document.addEventListener("keyup",e=>{
 if(e.key==="ArrowLeft" || e.key.toLowerCase()==="a") {
  keys.left=false;
 }

 if(e.key==="ArrowRight" || e.key.toLowerCase()==="d") {
  keys.right=false;
 }
});

window.addEventListener("blur",()=>{
 keys.left=false;
 keys.right=false;
 stopCanvasSteering();
});

drawBackground();
</script>
</body>
</html>
