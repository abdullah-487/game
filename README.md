<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1,user-scalable=no"/>
<title>Taken Fighters V3 (Mobile)</title>
<style>
  html,body{margin:0;height:100%;background:#0b1220;color:#e6eef8;font-family:Inter,system-ui;overflow:hidden;}
  canvas{display:block;margin:0 auto;background:radial-gradient(circle at center,#0b1220,#060a12);border-radius:10px;box-shadow:0 0 24px rgba(0,0,0,0.7);}
  #hud{position:fixed;top:10px;left:10px;font-size:14px;color:#7dd3fc;}
  /* Mobile buttons */
  #controls{position:fixed;bottom:10px;left:10px;width:100%;display:flex;justify-content:space-between;pointer-events:none;}
  .btn{pointer-events:auto;user-select:none;background:#1e293b;color:#e2e8f0;border:none;border-radius:50%;width:60px;height:60px;font-size:20px;margin:8px;box-shadow:0 0 10px rgba(0,0,0,0.5);}
  #right-buttons{position:fixed;bottom:10px;right:10px;display:flex;flex-direction:column;}
  #joystick{position:fixed;bottom:10px;left:10px;width:120px;height:120px;background:rgba(255,255,255,0.05);border-radius:50%;touch-action:none;}
  #stick{position:absolute;left:40px;top:40px;width:40px;height:40px;background:rgba(125,211,252,0.6);border-radius:50%;}
</style>
</head>
<body>
<canvas id="game" width="900" height="550"></canvas>
<div id="hud"></div>

<!-- MOBILE CONTROLS -->
<div id="joystick"><div id="stick"></div></div>
<div id="right-buttons">
  <button class="btn" id="shoot">🔥</button>
  <button class="btn" id="weapon">🔫</button>
  <button class="btn" id="mode">🫡</button>
</div>

<script>
const canvas=document.getElementById('game'),ctx=canvas.getContext('2d');
let W=canvas.width,H=canvas.height,keys={},last=0,gameRunning=true;

const ac=new (window.AudioContext||window.webkitAudioContext)();
function sound(freq,dur,type='square',vol=0.1){
  const o=ac.createOscillator(),g=ac.createGain();
  o.type=type;o.frequency.value=freq;g.gain.value=vol;
  o.connect(g).connect(ac.destination);o.start();o.stop(ac.currentTime+dur);
}
function explode(x,y){for(let i=0;i<20;i++)particles.push({x,y,vx:(Math.random()*2-1)*200,vy:(Math.random()*2-1)*200,life:0.5+Math.random()*0.5});sound(60,0.2,'sawtooth',0.2);}

class Entity{constructor(x,y,r){this.x=x;this.y=y;this.vx=0;this.vy=0;this.r=r;this.dead=false;this.hp=this.maxhp=1;}
  dist(o){return Math.hypot(this.x-o.x,this.y-o.y);}angle(o){return Math.atan2(o.y-this.y,o.x-this.x);}
}
class Bullet extends Entity{
  constructor(x,y,a,speed,fromAlly,damage){super(x,y,3);this.vx=Math.cos(a)*speed;this.vy=Math.sin(a)*speed;this.life=2;this.fromAlly=fromAlly;this.damage=damage;}
  update(dt){this.x+=this.vx*dt;this.y+=this.vy*dt;this.life-=dt;if(this.life<0)this.dead=true;}
}
class Player extends Entity{
  constructor(x,y){super(x,y,14);this.speed=180;this.lives=3;this.cool=0;this.weapon=0;}
  update(dt,ax=0,ay=0){
    const m=Math.hypot(ax,ay);if(m){ax/=m;ay/=m;}
    this.vx=ax*this.speed;this.vy=ay*this.speed;this.x+=this.vx*dt;this.y+=this.vy*dt;
    this.x=Math.max(this.r,Math.min(W-this.r,this.x));this.y=Math.max(this.r,Math.min(H-this.r,this.y));
    if(this.cool>0)this.cool-=dt;if(keys[' ']&&this.cool<=0){this.shoot();this.cool=this.weapon==1?0.8:0.25;}
  }
  shoot(){
    sound(300+Math.random()*50,0.1,'square',0.05);
    if(this.weapon==0){bullets.push(new Bullet(this.x,this.y,0,400,true,1));}
    else if(this.weapon==1){for(let i=-2;i<=2;i++)bullets.push(new Bullet(this.x,this.y,i*0.15,350,true,0.6));}
    else{bullets.push(new Bullet(this.x,this.y,0,700,true,2));sound(500,0.2,'sine',0.08);}
  }
}
class Fighter extends Entity{
  constructor(x,y,ally=true){super(x,y,12);this.ally=ally;this.speed=ally?120:140;this.shootT=1+Math.random();this.hp=this.maxhp=3;}
  update(dt){
    let target=null;
    if(this.ally){if(allyMode==='follow')target=player;else if(allyMode==='attack')target=enemies[0]||fighters.find(f=>!f.ally);}
    else target=player;
    if(target){
      const a=this.angle(target),d=this.dist(target);
      if(d>40){this.vx=Math.cos(a)*this.speed;this.vy=Math.sin(a)*this.speed;this.x+=this.vx*dt;this.y+=this.vy*dt;}
      this.shootT-=dt;if(this.shootT<0){this.shootT=1+Math.random();bullets.push(new Bullet(this.x,this.y,a,260,this.ally,1));sound(250,0.05,'square',0.04);}
    }
  }
}
class Enemy extends Entity{
  constructor(x,y){super(x,y,14);this.speed=100;this.hp=this.maxhp=2;}
  update(dt){const a=this.angle(player);this.vx=Math.cos(a)*this.speed;this.vy=Math.sin(a)*this.speed;this.x+=this.vx*dt;this.y+=this.vy*dt;}
}
class Hostage extends Entity{constructor(x,y){super(x,y,10);this.rescued=false;}}

let player=new Player(W/2,H/2),hostage=new Hostage(Math.random()*W,Math.random()*H);
let enemies=[],allies=[],fighters=[],bullets=[],particles=[];
for(let i=0;i<5;i++)enemies.push(new Enemy(Math.random()*W,Math.random()*H));
for(let i=0;i<3;i++)allies.push(new Fighter(Math.random()*W,Math.random()*H,true));
for(let i=0;i<3;i++)fighters.push(new Fighter(Math.random()*W,Math.random()*H,false));
let score=0,allyMode='follow';

window.addEventListener('keydown',e=>{
  keys[e.key]=true;
  if(e.key==='1')player.weapon=0;if(e.key==='2')player.weapon=1;if(e.key==='3')player.weapon=2;
  if(e.key==='f')allyMode='follow';if(e.key==='g')allyMode='attack';
});
window.addEventListener('keyup',e=>keys[e.key]=false);

function update(dt,ax=0,ay=0){
  player.update(dt,ax,ay);
  enemies.forEach(e=>e.update(dt));allies.forEach(a=>a.update(dt));fighters.forEach(f=>f.update(dt));
  bullets.forEach(b=>b.update(dt));
  particles.forEach(p=>{p.x+=p.vx*dt;p.y+=p.vy*dt;p.life-=dt;});particles=particles.filter(p=>p.life>0);

  bullets.forEach(b=>{
    if(b.fromAlly){
      [...enemies,...fighters.filter(f=>!f.ally)].forEach(e=>{
        if(!e.dead&&b.dist(e)<e.r+4){e.hp-=b.damage;b.dead=true;if(e.hp<=0){e.dead=true;explode(e.x,e.y);score+=10;}}
      });
    }else{
      if(b.dist(player)<player.r+4){b.dead=true;player.lives--;sound(80,0.2,'triangle',0.15);if(player.lives<=0)gameRunning=false;}
      allies.forEach(a=>{if(!a.dead&&b.dist(a)<a.r+4){a.hp--;b.dead=true;if(a.hp<=0){a.dead=true;explode(a.x,a.y);}}});
    }
  });
  bullets=bullets.filter(b=>!b.dead);
  enemies=enemies.filter(e=>!e.dead);
  fighters=fighters.filter(f=>!f.dead);
  allies=allies.filter(a=>!a.dead);
  if(player.dist(hostage)<25&&!hostage.rescued){hostage.rescued=true;score+=200;sound(600,0.4,'sine',0.15);}
}

function drawHealth(x,y,hp,maxhp,color){
  ctx.fillStyle=color;ctx.fillRect(x-10,y-20,hp/maxhp*20,3);ctx.strokeStyle='#000';ctx.strokeRect(x-10,y-20,20,3);
}
function render(){
  ctx.clearRect(0,0,W,H);
  ctx.fillStyle='rgba(255,255,255,0.03)';for(let x=0;x<W;x+=40){ctx.fillRect(x,0,1,H);}for(let y=0;y<H;y+=40){ctx.fillRect(0,y,W,1);}
  ctx.fillStyle='#34d399';ctx.beginPath();ctx.arc(hostage.x,hostage.y,hostage.r,0,Math.PI*2);ctx.fill();
  ctx.fillStyle='#7dd3fc';ctx.beginPath();ctx.arc(player.x,player.y,player.r,0,Math.PI*2);ctx.fill();
  drawHealth(player.x,player.y,player.lives,3,'#7dd3fc');
  ctx.fillStyle='#fb7185';enemies.forEach(e=>{ctx.beginPath();ctx.arc(e.x,e.y,e.r,0,Math.PI*2);ctx.fill();drawHealth(e.x,e.y,e.hp,e.maxhp,'#fb7185');});
  ctx.fillStyle='#22c55e';allies.forEach(a=>{ctx.beginPath();ctx.arc(a.x,a.y,a.r,0,Math.PI*2);ctx.fill();drawHealth(a.x,a.y,a.hp,a.maxhp,'#22c55e');});
  ctx.fillStyle='#f43f5e';fighters.forEach(f=>{ctx.beginPath();ctx.arc(f.x,f.y,f.r,0,Math.PI*2);ctx.fill();drawHealth(f.x,f.y,f.hp,f.maxhp,'#f43f5e');});
  ctx.fillStyle='#e2e8f0';bullets.forEach(b=>{ctx.beginPath();ctx.arc(b.x,b.y,3,0,Math.PI*2);ctx.fill();});
  particles.forEach(p=>{ctx.fillStyle=`rgba(255,200,80,${p.life})`;ctx.fillRect(p.x,p.y,3,3);});
  document.getElementById('hud').textContent=`Score:${score} | Lives:${player.lives} | Mode:${allyMode.toUpperCase()} | Weapon:${['Pistol','Shotgun','Laser'][player.weapon]}`;
}

let joystick=document.getElementById('joystick'),stick=document.getElementById('stick');
let joyX=0,joyY=0,isDragging=false;
function setJoystick(e){
  const rect=joystick.getBoundingClientRect();
  const touch=e.touches?e.touches[0]:e;
  const x=touch.clientX-rect.left-rect.width/2,y=touch.clientY-rect.top-rect.height/2;
  const d=Math.hypot(x,y),max=40;
  const clampedD=Math.min(max,d);
  const angle=Math.atan2(y,x);
  joyX=Math.cos(angle)*clampedD/max;
  joyY=Math.sin(angle)*clampedD/max;
  stick.style.left=40+joyX*40+'px';
  stick.style.top=40+joyY*40+'px';
}
function resetJoystick(){joyX=joyY=0;stick.style.left='40px';stick.style.top='40px';}
joystick.addEventListener('touchstart',e=>{isDragging=true;setJoystick(e);});
joystick.addEventListener('touchmove',e=>{if(isDragging)setJoystick(e);});
joystick.addEventListener('touchend',()=>{isDragging=false;resetJoystick();});
joystick.addEventListener('mousedown',e=>{isDragging=true;setJoystick(e);});
window.addEventListener('mousemove',e=>{if(isDragging)setJoystick(e);});
window.addEventListener('mouseup',()=>{isDragging=false;resetJoystick();});

document.getElementById('shoot').addEventListener('touchstart',()=>{keys[' ']=true;player.shoot();});
document.getElementById('shoot').addEventListener('touchend',()=>{keys[' ']=false;});
document.getElementById('weapon').addEventListener('click',()=>{player.weapon=(player.weapon+1)%3;sound(400,0.1,'triangle',0.1);});
document.getElementById('mode').addEventListener('click',()=>{allyMode=allyMode==='follow'?'attack':'follow';sound(200,0.1,'square',0.1);});

function loop(ts){const dt=(ts-last)/1000;last=ts;if(gameRunning){update(dt,joyX,joyY);render();requestAnimationFrame(loop);}else{ctx.fillStyle='#f87171';ctx.font='30px Arial';ctx.fillText('GAME OVER',W/2-100,H/2);}}
requestAnimationFrame(loop);
</script>
</body>
</html>
