<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>เกมหลบลูกบอล + AoE Gun</title>
<style>
:root{--bg:#f5f5f5;--panel:#fff;--ink:#111;}
*{box-sizing:border-box;}
body{margin:0;background:var(--bg);font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial;color:var(--ink);display:grid;place-items:center;min-height:100vh;padding:16px;}
.wrap{width:420px;max-width:100%;}
.panel{background:var(--panel);border:2px solid #000;border-radius:12px;padding:10px;box-shadow:0 10px 20px rgba(0,0,0,.06);}
.row{display:flex;gap:8px;align-items:center;justify-content:space-between;flex-wrap:wrap;margin-bottom:8px;}
.hint{font-size:10px;opacity:.75;}
.badge{padding:2px 8px;border-radius:999px;border:1px solid #000;font-size:12px;}
.ok{background:#e8f7ec;}.warn{background:#fff7ed;}.danger{background:#fee2e2;}
button,label.input{border:1px solid #000;border-radius:8px;padding:6px 10px;background:#fff;cursor:pointer;}
input[type=file]{display:none;}
canvas{display:block;width:100%;height:auto;background:#fff;border-radius:8px;cursor:pointer;}
.footer{text-align:center;font-size:10px;margin-top:8px;opacity:.7;}
#hudScore{font-weight:bold;font-size:16px;color:#facc15;}
</style>
</head>
<body>
<div class="wrap">
  <div class="panel row">
    <div>
      <strong>เกมหลบลูกบอล</strong>
      <div class="hint">ลูกบอลแดงต้องหลบ • Items & Gun AoE • กด Space เริ่ม/Restart</div>
    </div>
    <div class="row">
      <label class="input" for="file">อัปโหลดรูปตัวละคร (ไม่จำเป็น)</label>
      <input id="file" type="file" accept="image/*"/>
      <span id="imgState" class="badge warn">ยังไม่โหลดรูป</span>
    </div>
  </div>
  <canvas id="gameCanvas" width="400" height="500"></canvas>
  <div class="panel row" style="margin-top:8px">
    <div id="hudScore" class="badge">Score: 0</div>
    <div id="hudSpeed" class="badge">Speed: 3.0</div>
    <div id="hudBalls" class="badge">Balls: 0</div>
    <div id="hudGun" class="badge">Gun Lv:0 Ammo:10 CD:--</div>
  </div>
  <div class="footer">TIP: ลูกบอลแดงหลบ, Items 🛡️💣❤️💥, คลิกเมาส์ยิง AoE รอบตัว, กด A/D หรือ Arrow = เคลื่อนที่</div>
</div>
<script>
const canvas=document.getElementById("gameCanvas"),ctx=canvas.getContext("2d");
const hudScore=document.getElementById("hudScore"),hudSpeed=document.getElementById("hudSpeed"),hudBalls=document.getElementById("hudBalls"),hudGun=document.getElementById("hudGun"),imgState=document.getElementById("imgState");

// Player image
const playerImg=new Image();
const fileInput=document.getElementById("file");
fileInput.addEventListener("change",(e)=>{
  const file=e.target.files[0]; if(!file) return;
  const url=URL.createObjectURL(file); loadPlayerImage(url);
});
const player={x:200-25,y:380,width:70,height:120,speed:6,ready:false,error:false,facing:1,walkPhase:0,shield:false,gunLevel:0,ammo:10,gunCD:0};

// Load player image
function loadPlayerImage(src){
  player.ready=false; player.error=false;
  imgState.textContent="กำลังโหลดรูป…"; imgState.className="badge warn";
  playerImg.crossOrigin="anonymous";
  playerImg.onload=()=>{player.ready=true; player.error=false; imgState.textContent="โหลดรูปสำเร็จ"; imgState.className="badge ok";}
  playerImg.onerror=()=>{player.ready=false; player.error=true; imgState.textContent="โหลดรูปไม่สำเร็จ (ใช้ตัวละครเริ่มต้น)"; imgState.className="badge danger";}
  playerImg.src=src;
}

// Game state
let balls=[],items=[],bullets=[],score=0,ballSpeed=3,frameCount=0,hp=5,maxHP=5,hitFlash=0,shieldTime=0,itemCooldown=0,gameOver=true,startScreen=true;

// Input
let leftPressed=false,rightPressed=false;
addEventListener("keydown",e=>{
  if(e.key==="ArrowLeft"||e.key==="a") leftPressed=true;
  if(e.key==="ArrowRight"||e.key==="d") rightPressed=true;
  if(e.code==="Space"&&(startScreen||gameOver)) {startScreen=false; restartGame();}
});
addEventListener("keyup",e=>{
  if(e.key==="ArrowLeft"||e.key==="a") leftPressed=false;
  if(e.key==="ArrowRight"||e.key==="d") rightPressed=false;
});

// Mouse click AoE
canvas.addEventListener("click",()=>{
  if(gameOver||startScreen) return;
  if(player.gunCD>0||player.ammo<=0) return;
  player.ammo--; player.gunCD=Math.max(30,300-30*player.gunLevel);
  const AoERadius=150;
  for(let i=balls.length-1;i>=0;i--){
    const b=balls[i];
    const dx=b.x+b.size/2-(player.x+player.width/2);
    const dy=b.y+b.size/2-(player.y+player.height/2);
    if(Math.sqrt(dx*dx+dy*dy)<=AoERadius){
      balls.splice(i,1); score++;
      bullets.push({x1:player.x+player.width/2,y1:player.y+player.height/2,x2:b.x+b.size/2,y2:b.y+b.size/2,life:15});
    }
  }
});

// Spawn balls
const MIN_SIZE=14,MAX_SIZE=34;
function spawnBall(){
  const size=Math.floor(Math.random()*(MAX_SIZE-MIN_SIZE+1))+MIN_SIZE;
  const x=Math.random()*(canvas.width-size);
  const y=-size;
  const speed=ballSpeed+(36-size)/10;
  balls.push({x,y,size,speed});
}

// Spawn items
function spawnItem(){
  if(itemCooldown>0){itemCooldown--; return;}
  if(Math.random()<0.01){
    const types=["shield","bomb","health","ammo"];
    const type=types[Math.floor(Math.random()*types.length)];
    const size=24,x=Math.random()*(canvas.width-size),y=-size;
    items.push({type,x,y,size});
    itemCooldown=100;
  }
}

// Player hitbox
function getPlayerHitbox(){const mX=15,mT=20,mB=10; return {x:player.x+mX,y:player.y+mT,width:player.width-mX*2,height:player.height-(mT+mB)}}

// Update
function update(){
  if(gameOver||startScreen) return;
  frameCount++;
  // move
  let moving=false;
  if(leftPressed && player.x>0){player.x-=player.speed; moving=true; player.facing=-1;}
  if(rightPressed && player.x+player.width<canvas.width){player.x+=player.speed; moving=true; player.facing=1;}
  if(moving) player.walkPhase+=0.35; else player.walkPhase*=0.8;

  // spawn
  if(Math.random()<0.03) spawnBall();
  spawnItem();

  // balls
  for(let i=balls.length-1;i>=0;i--){
    const b=balls[i]; b.y+=b.speed;
    const hb=getPlayerHitbox();
    if(!player.shield && hb.x<b.x+b.size && hb.x+hb.width>b.x && hb.y<b.y+b.size && hb.y+hb.height>b.y){
      hp--; hitFlash=6; if(hp<=0) gameOver=true;
      balls.splice(i,1);
    } else if(player.shield && hb.x<b.x+b.size && hb.x+hb.width>b.x && hb.y<b.y+b.size && hb.y+hb.height>b.y){
      balls.splice(i,1);
    }
    if(b.y>canvas.height) {balls.splice(i,1); score++; if(score%50===0 && player.ammo<25) player.ammo++;}
  }

  // items
  for(let i=items.length-1;i>=0;i--){
    const it=items[i]; it.y+=3;
    const hb=getPlayerHitbox();
    if(hb.x<it.x+it.size && hb.x+hb.width>it.x && hb.y<it.y+it.size && hb.y+hb.height>it.y){
      switch(it.type){
        case"shield": player.shield=true; shieldTime=300; break;
        case"bomb": balls=[]; score+=5; break;
        case"health": hp=Math.min(maxHP,hp+1); break;
        case"ammo": player.ammo=Math.min(25,player.ammo+3); break;
      }
      items.splice(i,1);
    } else if(it.y>canvas.height) items.splice(i,1);
  }

  // Shield countdown
  if(player.shield){shieldTime--; if(shieldTime<=0){player.shield=false; shieldTime=0;}}

  // Gun cooldown
  if(player.gunCD>0) player.gunCD--;

  // ball speed up every 5s
  if(frameCount%(60*5)===0) ballSpeed+=0.5;

  // Gun level
  player.gunLevel=Math.min(10,Math.floor(score/50));

  // HUD
  hudScore.textContent=`Score: ${score}`;
  hudSpeed.textContent=`Speed: ${ballSpeed.toFixed(1)}`;
  hudBalls.textContent=`Balls: ${balls.length}`;
  hudGun.textContent=`Gun Lv:${player.gunLevel} Ammo:${player.ammo} CD:${player.gunCD>0?player.gunCD:"--"}`;
}

// Draw
function drawPlaceholderCharacter(x,y,w,h){
  ctx.save(); ctx.translate(x+w/2,y+h/2); const step=Math.sin(player.walkPhase)*3; ctx.translate(0,step);
  ctx.fillStyle="#2b2b2b"; ctx.strokeStyle="#000"; ctx.lineWidth=2;
  ctx.fillRect(-w*0.25,-h*0.2,w*0.5,h*0.6);
  ctx.beginPath(); ctx.arc(0,-h*0.4,w*0.18,0,Math.PI*2); ctx.fill(); ctx.restore();
}
function drawPlayer(){
  const step=Math.sin(player.walkPhase)*3;
  if(player.ready && playerImg.naturalWidth>0){
    ctx.save();
    ctx.translate(player.x,player.y+step);
    if(player.facing===-1) ctx.scale(-1,1),ctx.translate(-player.width,0);
    ctx.drawImage(playerImg,0,0,player.width,player.height);
    ctx.restore();
  }else drawPlaceholderCharacter(player.x,player.y,player.width,player.height);

  if(player.shield){ctx.save(); ctx.strokeStyle="#22c55e"; ctx.lineWidth=4;
    ctx.beginPath(); ctx.ellipse(player.x+player.width/2,player.y+player.height/2,player.width*0.6,player.height*0.6,0,0,Math.PI*2); ctx.stroke(); ctx.restore();}
}

// HP
function drawHP(){const barW=200,barH=20; ctx.fillStyle="#000"; ctx.fillRect(10,10,barW,barH); ctx.fillStyle="#ef4444";
let drawHPWidth=(hp/maxHP)*barW; if(hitFlash%2===1) drawHPWidth=0; hitFlash--; ctx.fillRect(10,10,drawHPWidth,barH);
ctx.strokeStyle="#000"; ctx.strokeRect(10,10,barW,barH);}

// Balls
function drawBalls(){ctx.fillStyle="#ef4444"; for(const b of balls){ctx.beginPath(); ctx.arc(b.x+b.size/2,b.y+b.size/2,b.size/2,0,Math.PI*2); ctx.fill();}}

// Items
function drawItems(){ctx.font="24px Arial"; ctx.textAlign="center"; ctx.textBaseline="middle";
for(const it of items){let emoji=""; switch(it.type){case"shield":emoji="🛡️";break;case"bomb":emoji="💣";break;case"health":emoji="❤️";break;case"ammo":emoji="💥";break;}
ctx.fillText(emoji,it.x+it.size/2,it.y+it.size/2);}}

// Bullets AoE
function drawBullets(){for(let i=bullets.length-1;i>=0;i--){const b=bullets[i];
ctx.strokeStyle="yellow"; ctx.lineWidth=2; ctx.beginPath(); ctx.moveTo(b.x1,b.y1); ctx.lineTo(b.x2,b.y2); ctx.stroke(); b.life--; if(b.life<=0) bullets.splice(i,1);}}

// Draw start screen
function drawStartScreen(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle="#000"; ctx.font="16px Arial"; ctx.textAlign="center";
  ctx.fillText("เกมหลบลูกบอล",canvas.width/2,100);
  ctx.font="12px Arial";
  ctx.fillText("Controls: Arrow/A/D = move, Click = AoE Gun, Space = Start",canvas.width/2,130);
  ctx.fillText("Red Ball = โจมตี, Shield 🛡️, Bomb 💣, Health ❤️, Ammo 💥",canvas.width/2,150);
  ctx.fillText("Press SPACE to Start",canvas.width/2,170);
}

// Restart
function restartGame(){balls=[];items=[];bullets=[];ballSpeed=3;score=0;frameCount=0;
hp=maxHP; hitFlash=0;shieldTime=0;itemCooldown=0;
player.gunLevel=0; player.ammo=10; player.gunCD=0;
gameOver=false; player.x=canvas.width/2-player.width/2; player.facing=1; player.walkPhase=0;}

// Loop
function loop(){if(startScreen) drawStartScreen(); else{update(); ctx.clearRect(0,0,canvas.width,canvas.height); drawPlayer(); drawBalls(); drawItems(); drawBullets(); drawHP();
if(gameOver){ctx.fillStyle="#000";ctx.font="28px Arial";ctx.textAlign="center";ctx.fillText("Game Over!",canvas.width/2,canvas.height/2-8);
ctx.font="16px Arial"; ctx.fillText("Press SPACE to Restart",canvas.width/2,canvas.height/2+20);} }
requestAnimationFrame(loop);}
loop();
</script>
</body>
</html>
