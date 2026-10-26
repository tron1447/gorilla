
<!doctype html>
<html lang="et">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>Monkey Tag VR</title>
<style>
*{box-sizing:border-box}
body{margin:0;overflow:hidden;background:#10291c;color:white;font-family:Arial}
#menu{position:fixed;inset:0;z-index:10;display:grid;place-items:center;background:linear-gradient(#062519cc,#06130fee);text-align:center}
.panel{width:min(400px,92vw);padding:24px;border:2px solid #71ff9b;border-radius:22px;background:#153c2a;box-shadow:0 0 35px #45ff8655}
h1{font-size:clamp(28px,7vw,42px);margin:0 0 8px;color:#80ffa4}
button{border:0;border-radius:12px;background:#6cf492;color:#10291c;font-weight:bold;font-size:16px;padding:13px 17px;margin:5px;cursor:pointer}
button:active{transform:scale(.97)}
#status{font-size:13px;color:#c6ffda;min-height:18px}
#hud{position:fixed;z-index:3;top:12px;left:12px;text-shadow:1px 2px 3px #000;pointer-events:none;line-height:1.6}
#help{position:fixed;z-index:3;left:12px;bottom:10px;font-size:12px;text-shadow:1px 2px 3px #000;pointer-events:none}
#look{position:fixed;inset:0;z-index:1;touch-action:none}
#mobile{display:none;position:fixed;z-index:4;left:20px;bottom:25px}
#joy{width:110px;height:110px;border-radius:50%;border:2px solid #ffffff88;background:#ffffff30;position:relative;touch-action:none}
#stick{position:absolute;left:32px;top:32px;width:42px;height:42px;border-radius:50%;background:#7aff9f}
#jump{display:none;position:fixed;z-index:4;bottom:42px;right:25px;border-radius:50%;width:90px;height:90px}
#vrbutton{position:fixed;z-index:8;bottom:12px;left:50%;transform:translateX(-50%)}
</style>
</head>
<body>
<div id="menu">
 <div class="panel">
  <h1>🐒 MONKEY TAG VR</h1>
  <p>Põgene robotite eest, kogu münte ja avasta džunglit!</p>
  <button id="play">▶ MÄNGI</button>
  <button id="vr">🥽 MÄNGI VR-IS</button>
  <p id="status">Laadin mängu...</p>
  <p style="font-size:12px">VR-i jaoks ava mäng HTTPS-i kaudu Meta Quest 2 brauseris.</p>
 </div>
</div>
<div id="hud">🐒 <b>MONKEY TAG</b><br>
Mündid: <span id="coins">0</span><br>
Robotid: <span id="robots">4</span><br>
Tabamused: <span id="hits">0</span></div>
<div id="help">WASD: liikumine · Hiir: vaade · Tühik: hüpe<br>VR: juhtpulga kallutamine · päästik: hüpe</div>
<div id="look"></div>
<div id="mobile"><div id="joy"><div id="stick"></div></div></div>
<button id="jump">HÜPPA</button>

<script type="importmap">
{"imports":{"three":"https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js"}}
</script>
<script type="module">
import * as THREE from 'three';
import {VRButton} from 'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/webxr/VRButton.js';

const $=s=>document.querySelector(s);
const status=$('#status');
const scene=new THREE.Scene();
scene.background=new THREE.Color(0x8edbc6);
scene.fog=new THREE.Fog(0x8edbc6,35,100);

const camera=new THREE.PerspectiveCamera(75,innerWidth/innerHeight,.1,180);
const renderer=new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth,innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio,1.7));
renderer.xr.enabled=true;
renderer.shadowMap.enabled=true;
renderer.shadowMap.type=THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

scene.add(new THREE.HemisphereLight(0xe3fff2,0x38502d,2.2));
const sun=new THREE.DirectionalLight(0xfff0c5,2.4);
sun.position.set(15,25,12);
sun.castShadow=true;
sun.shadow.mapSize.set(1024,1024);
scene.add(sun);

const material=(c,r=.85)=>new THREE.MeshStandardMaterial({color:c,roughness:r});
const grass=material(0x3b9d50),darkGrass=material(0x246c39),
wood=material(0x79502e),leaf=material(0x168348),
leaf2=material(0x36b958),stone=material(0x788b88),
gold=material(0xffd447,.35),robotMetal=material(0x9cabbc,.45),
red=material(0xff4545,.3),white=material(0xffffff),
black=material(0x19232a),brown=material(0x85522e);

function box(parent,x,y,z,w,h,d,m){
 const o=new THREE.Mesh(new THREE.BoxGeometry(w,h,d),m);
 o.position.set(x,y,z);o.castShadow=true;o.receiveShadow=true;
 parent.add(o);return o;
}
function ball(parent,x,y,z,r,m){
 const o=new THREE.Mesh(new THREE.SphereGeometry(r,14,10),m);
 o.position.set(x,y,z);o.castShadow=true;parent.add(o);return o;
}
function cyl(parent,x,y,z,r1,r2,h,m){
 const o=new THREE.Mesh(new THREE.CylinderGeometry(r1,r2,h,12),m);
 o.position.set(x,y,z);o.castShadow=true;parent.add(o);return o;
}

// Jungle floor and boundary
box(scene,0,-.8,0,48,1.6,48,grass);
box(scene,0,-1.7,0,42,1.2,42,darkGrass);

// Trees
for(let i=0;i<42;i++){
 const x=(Math.random()-.5)*41,z=(Math.random()-.5)*41;
 if(Math.abs(x)<5&&Math.abs(z)<5)continue;
 cyl(scene,x,.5,z,.2,.35,2.6,wood);
 ball(scene,x,2.1,z,1.05,Math.random()>.5?leaf:leaf2);
 ball(scene,x-.5,2.3,z+.1,.65,leaf);
 ball(scene,x+.45,2.55,z-.15,.6,leaf2);
}

// Parkour platforms
const platforms=[
 [0,1.1,-5,4, .4,4,wood],
 [-5,2.5,-8,3,.5,3,stone],
 [4,3.4,-10,3,.5,3,stone],
 [0,4.5,-14,4,.5,3,wood],
 [-8,1.8,0,3,.5,3,stone],
 [8,2.1,1,3,.5,3,stone],
 [0,1.8,8,4,.5,3,wood],
 [-12,.3,-17,2,.5,2,stone],
 [-8,.8,-17,2,.5,2,stone],
 [-4,1.3,-17,2,.5,2,stone],
 [4,1.2,12,3,.5,3,stone],
 [9,2.2,15,3,.5,3,stone]
];
for(const p of platforms)box(scene,...p.slice(0,3),...p.slice(3,6),
 material(p[6]===wood?0x79502e:0x788b88));

// Player rig
const player=new THREE.Group();
scene.add(player);
player.position.set(0,0,5);
const headRig=new THREE.Group();
player.add(headRig);
headRig.position.set(0,1.65,0);
headRig.add(camera);
camera.position.set(0,0,0);
camera.rotation.order='YXZ';

// Monkey avatar visible to the player in desktop mode
const monkey=new THREE.Group();
player.add(monkey);
ball(monkey,0,1.05,0,.35,brown);
ball(monkey,-.24,1.32,0,.15,brown);
ball(monkey,.24,1.32,0,.15,brown);
ball(monkey,-.12,1.1,-.31,.055,white);
ball(monkey,.12,1.1,-.31,.055,white);
ball(monkey,-.12,1.1,-.36,.025,black);
ball(monkey,.12,1.1,-.36,.025,black);
ball(monkey,0,.94,-.35,.08,brown);
monkey.visible=false;

// Robot enemies
const robots=[];
function makeRobot(x,z,type){
 const g=new THREE.Group();
 scene.add(g);g.position.set(x,0,z);
 box(g,0,.7,0,.68,.8,.55,robotMetal);
 ball(g,0,1.3,0,.39,robotMetal);
 ball(g,-.14,1.34,-.34,.085,red);
 ball(g,.14,1.34,-.34,.085,red);
 box(g,0,.27,0,.8,.15,.65,black);
 box(g,-.22,.05,0,.18,.15,.2,black);
 box(g,.22,.05,0,.18,.15,.2,black);
 cyl(g,0,1.75,0,.06,.06,.25,red);
 robots.push({g,type,speed:type===0?2.2:type===1?1.4:1.8,
  startX:x,startZ:z,phase:Math.random()*6,stun:0});
}
makeRobot(7,-4,0);
makeRobot(-7,-6,1);
makeRobot(10,8,2);
makeRobot(-10,5,0);

// Collectable coins
const coins=[];
for(let i=0;i<18;i++){
 const c=new THREE.Mesh(new THREE.TorusGeometry(.3,.09,8,16),gold);
 c.position.set((Math.random()-.5)*30,.8+Math.random()*1.5,(Math.random()-.5)*30);
 c.rotation.y=Math.PI/2;
 scene.add(c);coins.push({mesh:c,taken:false});
}

// Controls
const keys={};
let yaw=0,pitch=0,started=false;
let vertical=0,onGround=true,score=0,hits=0;
let joyX=0,joyY=0,joyId=null;
let lastTouchX=0,lastTouchY=0,touchLook=false;

document.addEventListener('keydown',e=>{
 keys[e.code]=true;
 if(e.code==='Space'){e.preventDefault();jump();}
});
document.addEventListener('keyup',e=>keys[e.code]=false);

renderer.domElement.addEventListener('click',()=>{
 if(started&&!renderer.xr.isPresenting&&innerWidth>800)
  renderer.domElement.requestPointerLock?.();
});
document.addEventListener('mousemove',e=>{
 if(document.pointerLockElement===renderer.domElement&&!renderer.xr.isPresenting){
  yaw-=e.movementX*.0025;
  pitch=THREE.MathUtils.clamp(pitch-e.movementY*.0025,-1.35,1.35);
  camera.rotation.set(pitch,yaw,0,'YXZ');
 }
});

// Touch camera look
const look=$('#look');
look.addEventListener('touchstart',e=>{
 if(e.target.closest('#joy')||e.target.closest('button'))return;
 const t=e.changedTouches[0];
 lastTouchX=t.clientX;lastTouchY=t.clientY;touchLook=true;
},{passive:true});
look.addEventListener('touchmove',e=>{
 if(!touchLook)return;
 const t=e.changedTouches[0];
 yaw-=(t.clientX-lastTouchX)*.004;
 pitch=THREE.MathUtils.clamp(pitch-(t.clientY-lastTouchY)*.004,-1.3,1.3);
 lastTouchX=t.clientX;lastTouchY=t.clientY;
 camera.rotation.set(pitch,yaw,0,'YXZ');
},{passive:true});
look.addEventListener('touchend',()=>touchLook=false);

function jump(){if(onGround){vertical=6.8;onGround=false;}}

// Mobile joystick
const joy=$('#joy'),stick=$('#stick');
joy.addEventListener('pointerdown',e=>{
 joyId=e.pointerId;joy.setPointerCapture(e.pointerId);moveJoy(e);
});
joy.addEventListener('pointermove',e=>{if(e.pointerId===joyId)moveJoy(e);});
function moveJoy(e){
 const r=joy.getBoundingClientRect();
 let dx=e.clientX-(r.left+r.width/2),dy=e.clientY-(r.top+r.height/2);
 const d=Math.hypot(dx,dy),max=33;
 if(d>max){dx*=max/d;dy*=max/d;}
 joyX=dx/max;joyY=dy/max;
 stick.style.left=(32+dx)+'px';stick.style.top=(32+dy)+'px';
}
function stopJoy(){
 joyId=null;joyX=joyY=0;
 stick.style.left='32px';stick.style.top='32px';
}
joy.addEventListener('pointerup',stopJoy);
joy.addEventListener('pointercancel',stopJoy);
$('#jump').addEventListener('pointerdown',jump);

// Start
function startGame(mobile=false){
 started=true;$('#menu').style.display='none';
 if(mobile){$('#mobile').style.display='block';$('#jump').style.display='block';}
}
$('#play').onclick=()=>startGame(innerWidth<900);

const vrButton=VRButton.createButton(renderer);
vrButton.id='vrbutton';
vrButton.style.display='none';
document.body.appendChild(vrButton);
$('#vr').onclick=async()=>{
 if(!navigator.xr){
  status.textContent='WebXR pole saadaval. Ava mäng Meta Quest 2 brauseris HTTPS-i kaudu.';
  return;
 }
 try{
  const ok=await navigator.xr.isSessionSupported('immersive-vr');
  if(!ok){status.textContent='See brauser ei toeta VR-režiimi.';return;}
  startGame(false);
  vrButton.click();
 }catch(e){status.textContent='VR-i käivitamine ebaõnnestus: '+e.message;}
};

// VR controller trigger jumps; thumbstick moves the player
const controllers=[];
for(let i=0;i<2;i++){
 const c=renderer.xr.getController(i);
 c.addEventListener('selectstart',jump);
 scene.add(c);controllers.push(c);
}
function getVRMove(){
 let x=0,z=0;
 for(const c of controllers){
  const gp=c.userData.gamepad;
  if(!gp)continue;
  const axes=gp.axes||[];
  if(axes.length>=4){
   x+=Math.abs(axes[2])>.15?axes[2]:0;
   z+=Math.abs(axes[3])>.15?axes[3]:0;
  }else if(axes.length>=2){
   x+=Math.abs(axes[0])>.15?axes[0]:0;
   z+=Math.abs(axes[1])>.15?axes[1]:0;
  }
 }
 return {x:THREE.MathUtils.clamp(x,-1,1),z:THREE.MathUtils.clamp(z,-1,1)};
}
renderer.xr.addEventListener('sessionstart',()=>{
 status.textContent='VR ühendatud!';
 $('#help').textContent='VR: kalluta juhtpulka liikumiseks ja vajuta päästikut hüppamiseks';
});
renderer.xr.addEventListener('sessionend',()=>{status.textContent='VR-seanss lõppes.';});

// Movement, robot AI, coins
const clock=new THREE.Clock();
const forward=new THREE.Vector3(),right=new THREE.Vector3(),move=new THREE.Vector3();
function animate(){
 const dt=Math.min(clock.getDelta(),.04);
 const vr=renderer.xr.isPresenting;
 const axes=vr?getVRMove():{x:(keys.KeyD?1:0)-(keys.KeyA?1:0)+joyX,
  z:(keys.KeyS?1:0)-(keys.KeyW?1:0)+joyY};
 if(started){
  // In VR, use headset viewing direction to decide where forward is.
  camera.getWorldDirection(forward);
  forward.y=0;
  if(forward.lengthSq()<.001)forward.set(0,0,-1);
  forward.normalize();
  right.set(-forward.z,0,forward.x);
  move.set(0,0,0);
  move.addScaledVector(right,axes.x);
  move.addScaledVector(forward,-axes.z);
  if(move.lengthSq()>1)move.normalize();

  const speed=vr?3.2:5.5;
  player.position.addScaledVector(move,speed*dt);

  if(!vr){
   if(keys.ArrowLeft)yaw+=dt*1.8;
   if(keys.ArrowRight)yaw-=dt*1.8;
   camera.rotation.set(pitch,yaw,0,'YXZ');
  }

  vertical-=17*dt;
  player.position.y+=vertical*dt;
  if(player.position.y<=0){player.position.y=0;vertical=0;onGround=true;}

  // Keep player in the play area.
  player.position.x=THREE.MathUtils.clamp(player.position.x,-21,21);
  player.position.z=THREE.MathUtils.clamp(player.position.z,-21,21);

  // Simple platform landing
  const px=player.position.x,pz=player.position.z;
  for(const p of platforms){
   const [x,y,z,w,h,d]=p;
   const feet=player.position.y;
   if(px>x-w/2-.3&&px<x+w/2+.3&&pz>z-d/2-.3&&pz<z+d/2+.3&&
      vertical<=0&&feet>=y+h/2-.15&&feet<=y+h/2+.35){
    player.position.y=y+h/2;vertical=0;onGround=true;
   }
  }

  // Coins
  for(const c of coins){
   if(c.taken)continue;
   c.mesh.rotation.y+=dt*2;
   c.mesh.position.y+=Math.sin(clock.elapsedTime*2+c.mesh.position.x)*dt*.15;
   if(c.mesh.position.distanceTo(new THREE.Vector3(px,player.position.y+1,pz))<1){
    c.taken=true;scene.remove(c.mesh);score++;
    $('#coins').textContent=score;
   }
  }

  // Robots follow player. Blue-ish type wanders until player gets close.
  for(const r of robots){
   const g=r.g;
   if(r.stun>0){r.stun-=dt;continue;}
   const dx=px-g.position.x,dz=pz-g.position.z;
   const dist=Math.hypot(dx,dz);
   let tx=px,tz=pz;
   if(r.type===1&&dist>8){
    tx=r.startX+Math.sin(clock.elapsedTime+r.phase)*4;
    tz=r.startZ+Math.cos(clock.elapsedTime*.7+r.phase)*4;
   }
   const vx=tx-g.position.x,vz=tz-g.position.z,vd=Math.hypot(vx,vz);
   if(vd>.05){
    g.position.x+=vx/vd*r.speed*dt;
    g.position.z+=vz/vd*r.speed*dt;
    g.rotation.y=Math.atan2(vx,vz);
   }
   g.position.y=.08+Math.abs(Math.sin(clock.elapsedTime*4+r.phase))*.08;
   if(dist<1.05&&Math.abs(player.position.y)<1.4){
    hits++;$('#hits').textContent=hits;
    player.position.set(0,0,5);vertical=0;
    r.stun=1.5;
   }
  }
 }
 renderer.render(scene,camera);
}
renderer.setAnimationLoop(animate);

window.addEventListener('resize',()=>{
 camera.aspect=innerWidth/innerHeight;
 camera.updateProjectionMatrix();
 renderer.setSize(innerWidth,innerHeight);
});
status.textContent='Valmis! Vajuta MÄNGI või SISENE VR-I.';
</script>
</body>
</html>
