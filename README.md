[index (3).html](https://github.com/user-attachments/files/33952596/index.3.html)<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>Monkey Tag — VR Multiplayer</title>
<style>
  :root { color-scheme: dark; --panel:rgba(10,18,28,.88); --accent:#80ffb0; }
  * { box-sizing:border-box; }
  html,body { margin:0; width:100%; height:100%; overflow:hidden; font-family:system-ui,Arial,sans-serif; background:#87ceeb; }
  canvas { display:block; touch-action:none; }
  #menu { position:fixed; z-index:5; left:12px; top:12px; width:min(350px,calc(100vw - 24px)); padding:14px; border:1px solid #ffffff44; border-radius:16px; background:var(--panel); color:white; box-shadow:0 8px 30px #0005; }
  #menu.min>*:not(h1) { display:none; } #menu.min { width:auto; }
  h1 { margin:0 0 5px; font-size:22px; cursor:pointer; } p { margin:6px 0 10px; font-size:13px; line-height:1.35; color:#d5e4ef; }
  .line { display:flex; gap:7px; margin:8px 0; }
  input,button { min-width:0; border:0; border-radius:10px; padding:11px 12px; font:inherit; }
  input { flex:1; background:#f5f8fc; color:#15202b; }
  #room { text-transform:uppercase; }
  button { cursor:pointer; font-weight:750; background:var(--accent); color:#10251a; }
  button.secondary { background:#314454; color:white; }
  #status { font-size:12px; color:#bcebd0; min-height:17px; overflow-wrap:anywhere; }
  #help { position:fixed; z-index:3; right:12px; top:12px; max-width:260px; padding:10px 12px; border-radius:12px; background:#0a121cc9; color:white; font-size:12px; line-height:1.4; }
  #roomTag { font-weight:800; color:var(--accent); }
  #hud { position:fixed; z-index:3; left:50%; top:12px; transform:translateX(-50%); text-align:center; color:white; pointer-events:none; text-shadow:0 2px 6px #000a; width:min(520px,calc(100vw - 24px)); }
  #hud1 { display:inline-block; padding:8px 16px; border-radius:20px; background:#0a121cc9; font-weight:800; font-size:17px; }
  #hud2 { margin-top:6px; font-size:12px; }
  #mobile { display:none; position:fixed; z-index:4; left:0; right:0; bottom:14px; padding:0 18px; justify-content:space-between; align-items:end; pointer-events:none; }
  .pad { width:118px; height:118px; border-radius:50%; background:#14233277; border:2px solid #ffffff88; position:relative; pointer-events:auto; touch-action:none; }
  .nub { width:46px; height:46px; border-radius:50%; position:absolute; left:34px; top:34px; background:#e9fff0dd; border:2px solid #fff; }
  #jump { width:76px; height:76px; border-radius:50%; pointer-events:auto; touch-action:none; background:#80ffb0dd; border:2px solid white; }
  @media (pointer:coarse) { #mobile { display:flex; } #help { top:auto; bottom:145px; right:10px; max-width:175px; } #hud { top:60px; } }
</style>
</head>
<body>
<div id="menu">
  <h1 title="Show / hide menu">🐒 Monkey Tag ▾</h1>
  <p>One person is IT and chases the others. Touch a monkey to pass the tag on. Your score is the seconds you spend NOT being IT.</p>
  <div class="line"><input id="name" maxlength="12" placeholder="Your name"></div>
  <div class="line"><input id="room" maxlength="12" placeholder="ROOM CODE" value="BANANA"><button id="host">Create room</button></div>
  <div class="line"><button id="join" class="secondary" style="flex:1">Join room</button><button id="copy" class="secondary">Copy link</button><button id="vr" class="secondary">Help</button></div>
  <div id="status">Not connected. Choose Create room or Join room.</div>
  <p>Room: <span id="roomTag">—</span></p>
</div>
<div id="help">🖥️ WASD move, mouse look (click game first), Space jump, Shift sprint.<br>📱 Left pad to move, drag to look, JUMP button.<br>🥽 Join a room first, then Enter VR: left stick moves, right stick snap-turns, right trigger / A jumps.</div>
<div id="hud"><div id="hud1">🐒 Monkey Tag</div><div id="hud2"></div></div>
<div id="mobile"><div class="pad" id="pad"><div class="nub" id="nub"></div></div><button id="jump">JUMP</button></div>

<script src="https://cdn.jsdelivr.net/npm/peerjs@1.5.4/dist/peerjs.min.js"></script>
<script type="importmap">
{"imports":{"three":"https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js","three/addons/":"https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"}}
</script>
<script type="module">
import * as THREE from 'three';
import { VRButton } from 'three/addons/webxr/VRButton.js';

const $=s=>document.querySelector(s), clamp=THREE.MathUtils.clamp, r2=v=>Math.round(v*100)/100;
const statusEl=$('#status'), roomInput=$('#room'), nameIn=$('#name');
const setStatus=s=>statusEl.textContent=s;
const cleanRoom=s=>s.toUpperCase().replace(/[^A-Z0-9]/g,'').slice(0,12);
try{nameIn.value=localStorage.getItem('mt-name')||'';}catch(e){}
nameIn.addEventListener('input',()=>{try{localStorage.setItem('mt-name',nameIn.value);}catch(e){}});
const myName=()=>nameIn.value.trim().slice(0,12)||'Monkey';
let myId='me', peer=null, isHost=false, room='';
const conns=new Map(), others=new Map();
let game={it:null,coolUntil:0,s:{}};
const myColor=new THREE.Color().setHSL(Math.random(),.75,.5);
const now=()=>performance.now();

// ---------- Scene ----------
const scene=new THREE.Scene();
const sk=document.createElement('canvas'); sk.width=4; sk.height=256;
const sx=sk.getContext('2d'), gr=sx.createLinearGradient(0,0,0,256);
gr.addColorStop(0,'#3a8fe0'); gr.addColorStop(.5,'#8bd7ff'); gr.addColorStop(1,'#e8f8ff'); sx.fillStyle=gr; sx.fillRect(0,0,4,256);
const skyTex=new THREE.CanvasTexture(sk); skyTex.mapping=THREE.EquirectangularReflectionMapping; skyTex.colorSpace=THREE.SRGBColorSpace;
scene.background=skyTex; scene.fog=new THREE.Fog(0x8bd7ff,30,95);
const camera=new THREE.PerspectiveCamera(70,innerWidth/innerHeight,.05,150);
const rig=new THREE.Group(); scene.add(rig); rig.add(camera);
const renderer=new THREE.WebGLRenderer({antialias:true});
renderer.setPixelRatio(Math.min(devicePixelRatio,1.7)); renderer.setSize(innerWidth,innerHeight);
renderer.xr.enabled=true; renderer.shadowMap.enabled=true; renderer.shadowMap.type=THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);
const vrButton=VRButton.createButton(renderer);
Object.assign(vrButton.style,{position:'fixed',bottom:'12px',left:'50%',transform:'translateX(-50%)',zIndex:'4'});
document.body.appendChild(vrButton);

scene.add(new THREE.HemisphereLight(0xdff6ff,0x4c7547,2));
const sun=new THREE.DirectionalLight(0xffffff,2.2); sun.position.set(-10,18,8); sun.castShadow=true;
sun.shadow.mapSize.set(2048,2048); Object.assign(sun.shadow.camera,{left:-30,right:30,top:30,bottom:-30,near:1,far:60}); scene.add(sun);

const mat=(color,roughness=.85)=>new THREE.MeshStandardMaterial({color,roughness});
const ground=new THREE.Mesh(new THREE.CylinderGeometry(24,25,1.2,48),mat(0x45b94b)); ground.position.y=-.6; ground.receiveShadow=true; scene.add(ground);
const dirt=new THREE.Mesh(new THREE.CylinderGeometry(24,25,1.1,48),mat(0x80512e)); dirt.position.y=-1.7; scene.add(dirt);
// grass tufts
{const m=new THREE.InstancedMesh(new THREE.ConeGeometry(.06,.35,4),mat(0x2f9a3a),260),d=new THREE.Object3D();
 for(let i=0;i<260;i++){const a=Math.random()*6.283,r=Math.sqrt(Math.random())*23;d.position.set(Math.cos(a)*r,.15,Math.sin(a)*r);d.rotation.set((Math.random()-.5)*.4,Math.random()*3,0);d.updateMatrix();m.setMatrixAt(i,d.matrix);}
 scene.add(m);}
const trunkMat=mat(0x79502d), leafMat=mat(0x168d55);
function addTree(x,z,s=1){
  const g=new THREE.Group(); g.position.set(x,0,z); g.scale.setScalar(s);
  const t=new THREE.Mesh(new THREE.CylinderGeometry(.18,.28,2,7),trunkMat); t.position.y=1; t.castShadow=true; g.add(t);
  for(let i=0;i<3;i++){const c=new THREE.Mesh(new THREE.IcosahedronGeometry(1.05-i*.12,1),leafMat); c.position.set((i-1)*.18,2.1+i*.48,0); c.castShadow=true; g.add(c);}
  scene.add(g);
}
for(const [x,z,s] of [[-8,-7,1.3],[-12,2,1.1],[9,-9,1.2],[13,1,1.1],[5,13,1.4],[-7,12,1.1],[16,8,.9],[-17,-5,.9],[-15,-14,1],[18,-8,1.1]]) addTree(x,z,s);
for(let i=0;i<12;i++){
  const rock=new THREE.Mesh(new THREE.DodecahedronGeometry(.35+Math.random()*.45,0),mat(0x8996a0));
  const a=Math.random()*6.283,r=5+Math.random()*16; rock.position.set(Math.cos(a)*r,.05,Math.sin(a)*r); rock.rotation.set(Math.random(),Math.random(),Math.random()); rock.castShadow=true; scene.add(rock);
}
const platforms=[];
for(const [x,y,z,w,d] of [[-4,1.2,-3,3,2],[1,2.2,-7,3,2],[6,1.2,-4,3,2],[-7,1.8,4,3,2],[8,1.4,7,3,2],[4,2.6,5,3,2]]){
  const m=new THREE.Mesh(new THREE.BoxGeometry(w,.35,d),mat(0xffc857)); m.position.set(x,y,z); m.castShadow=m.receiveShadow=true; scene.add(m); platforms.push({x,y,z,w,d,top:y+.175});
}
const hoop=new THREE.Mesh(new THREE.TorusGeometry(1.2,.12,10,28),new THREE.MeshStandardMaterial({color:0xff5e75,emissive:0xff2040,emissiveIntensity:.35})); hoop.position.set(0,2.3,-10); hoop.rotation.x=Math.PI/2; scene.add(hoop);
const clouds=[]; const cloudMat=new THREE.MeshStandardMaterial({color:0xffffff,emissive:0xffffff,emissiveIntensity:.5,fog:false});
for(let i=0;i<7;i++){
  const c=new THREE.Group();
  for(let j=0;j<4;j++){const s=new THREE.Mesh(new THREE.SphereGeometry(2+Math.random()*1.5,10,8),cloudMat); s.position.set(j*2.6-4,Math.random(),Math.random()*1.5); s.scale.y=.6; c.add(s);}
  c.position.set(-60+i*20,32+Math.random()*10,-40+Math.random()*80); scene.add(c); clouds.push(c);
}

// ---------- Monkeys ----------
function makeLabel(){
  const cv=document.createElement('canvas'); cv.width=256; cv.height=64;
  const t=new THREE.CanvasTexture(cv); t.colorSpace=THREE.SRGBColorSpace;
  const s=new THREE.Sprite(new THREE.SpriteMaterial({map:t,transparent:true})); s.scale.set(1.6,.4,1); s.userData={cv,t}; return s;
}
function drawLabel(s,text){
  const {cv,t}=s.userData, x=cv.getContext('2d'); x.clearRect(0,0,256,64);
  x.fillStyle='#0009'; x.beginPath(); x.roundRect(8,8,240,48,24); x.fill();
  x.fillStyle='#fff'; x.font='bold 28px system-ui,sans-serif'; x.textAlign='center'; x.fillText(text,128,43); t.needsUpdate=true;
}
function makeMonkey(color){
  const g=new THREE.Group(), fur=mat(color), skin=mat(0xf2c9a0), dark=mat(0x17202b);
  const add=(geo,m,x,y,z,p=g)=>{const o=new THREE.Mesh(geo,m); o.position.set(x,y,z); o.castShadow=true; p.add(o); return o;};
  add(new THREE.CapsuleGeometry(.32,.5,4,10),fur,0,.8,0);
  add(new THREE.SphereGeometry(.24,10,8),skin,0,.78,-.17);
  const head=add(new THREE.SphereGeometry(.33,16,12),fur,0,1.45,0);
  add(new THREE.SphereGeometry(.2,12,10),skin,0,-.07,-.17,head);
  for(const s of [-1,1]){
    add(new THREE.SphereGeometry(.14,10,8),skin,s*.36,.02,0,head);
    add(new THREE.SphereGeometry(.05,8,8),dark,s*.12,.08,-.3,head);
  }
  const limb=(x,y,len,r)=>{const p=new THREE.Group(); p.position.set(x,y,0); add(new THREE.CapsuleGeometry(r,len,4,8),fur,0,-len/2-r/2,0,p); g.add(p); return p;};
  const arms=[limb(-.43,1.1,.4,.09),limb(.43,1.1,.4,.09)], legs=[limb(-.18,.5,.3,.1),limb(.18,.5,.3,.1)];
  const tail=add(new THREE.TorusGeometry(.28,.045,6,14,Math.PI),fur,0,.55,.3); tail.rotation.y=Math.PI/2;
  const mark=new THREE.Mesh(new THREE.ConeGeometry(.2,.4,8),new THREE.MeshStandardMaterial({color:0xff3344,emissive:0xff2233,emissiveIntensity:.9}));
  mark.rotation.x=Math.PI; mark.visible=false; g.add(mark);
  const label=makeLabel(); label.position.y=2.15; g.add(label);
  g.userData={arms,legs,mark,label}; return g;
}
function addOther(id,color){
  const g=makeMonkey(/^#[0-9a-f]{6}$/i.test(color)?color:'#ff8a40'); scene.add(g);
  const o={g,tp:new THREE.Vector3(),ty:0,seen:now(),name:''}; others.set(id,o); drawLabel(g.userData.label,id.slice(-4)); return o;
}
function removeOther(id){const o=others.get(id); if(o){scene.remove(o.g); others.delete(id); delete game.s[id];}}
function clearOthers(){for(const id of [...others.keys()])removeOther(id); game={it:null,coolUntil:0,s:{}};}
// VR hands
for(let i=0;i<2;i++){const c=renderer.xr.getController(i); c.add(new THREE.Mesh(new THREE.SphereGeometry(.06,12,10),mat(0x6b4426))); rig.add(c);}

// ---------- HUD ----------
const hudC=document.createElement('canvas'); hudC.width=512; hudC.height=96;
const hudT=new THREE.CanvasTexture(hudC); hudT.colorSpace=THREE.SRGBColorSpace;
const hudS=new THREE.Sprite(new THREE.SpriteMaterial({map:hudT,transparent:true,depthTest:false}));
hudS.scale.set(.9,.17,1); hudS.position.set(0,.34,-1.2); hudS.renderOrder=99; hudS.visible=false; camera.add(hudS);
const nameOf=id=>id===myId?myName():(others.get(id)?.name||id.slice(-4));
let lastHud='';
function refreshHud(){
  let l1;
  if(!peer) l1='Solo mode: create or join a room to play tag';
  else if(!game.it) l1=others.size?'Starting…':'Waiting for friends · room '+room;
  else{
    const left=Math.ceil((game.coolUntil-now())/1000);
    l1=(game.it===myId?'🔴 YOU ARE IT! Catch someone!':'🏃 RUN! '+nameOf(game.it)+' is IT')+(left>0?' ('+left+')':'');
  }
  const l2=Object.entries(game.s).sort((a,b)=>b[1]-a[1]).slice(0,5).map(([id,v])=>nameOf(id)+' '+v+'s').join('  ·  ');
  if(l1+l2===lastHud) return; lastHud=l1+l2;
  $('#hud1').textContent=l1; $('#hud2').textContent=l2;
  const x=hudC.getContext('2d'); x.clearRect(0,0,512,96); x.fillStyle='#0a121ccc'; x.beginPath(); x.roundRect(4,4,504,88,40); x.fill();
  x.fillStyle='#fff'; x.textAlign='center'; x.font='bold 30px system-ui,sans-serif'; x.fillText(l1,256,46); x.font='20px system-ui,sans-serif'; x.fillText(l2.slice(0,60),256,76); hudT.needsUpdate=true;
}
let ac; function beep(f){try{ac??=new AudioContext(); const o=ac.createOscillator(),g=ac.createGain(); o.type='square'; o.frequency.value=f; g.gain.value=.05; o.connect(g); g.connect(ac.destination); o.start(); o.stop(ac.currentTime+.15);}catch(e){}}

// ---------- Networking (host relays; host decides who is IT) ----------
function sendAll(str,except){for(const c of conns.values()) if(c!==except&&c.open) try{c.send(str);}catch(e){}}
function attach(c){
  const mine=peer; conns.set(c.peer,c);
  c.on('open',()=>{setStatus('Connected! Tag your friends.'); $('#menu').classList.add('min');});
  c.on('data',raw=>onPacket(raw,c));
  c.on('close',()=>{
    if(mine!==peer) return; conns.delete(c.peer);
    if(isHost){removeOther(c.peer); sendAll(JSON.stringify({t:'bye',id:c.peer}));}
    else{setStatus('The host left the room.'); clearOthers();}
  });
  c.on('error',()=>setStatus('Connection issue. Try again.'));
}
function onPacket(raw,c){
  let p; try{p=JSON.parse(raw);}catch(e){return;}
  if(p.t==='s'){
    if(p.id===myId||typeof p.id!=='string'||!Array.isArray(p.p)||p.p.length!==4||!p.p.every(Number.isFinite)) return;
    let o=others.get(p.id), fresh=!o; if(fresh) o=addOther(p.id,p.c);
    o.tp.set(p.p[0],p.p[1],p.p[2]); o.ty=p.p[3]; o.seen=now(); if(fresh) o.g.position.copy(o.tp);
    const n=String(p.n||'').slice(0,12); if(n&&n!==o.name){o.name=n; drawLabel(o.g.userData.label,n);}
    if(isHost) sendAll(raw,c);
  }else if(p.t==='g'&&!isHost){
    game.it=p.it; game.coolUntil=now()+(p.c||0); game.s=p.s||{}; if(p.k) beep(p.k===myId?200:700);
  }else if(p.t==='bye'&&!isHost) removeOther(p.id);
}
function hostTick(dt){
  const ids=[myId,...others.keys()], P=id=>id===myId?pos:others.get(id).tp; let k=0;
  if(ids.length<2) game.it=null;
  else{
    if(!game.it||!ids.includes(game.it)){game.it=ids[Math.random()*ids.length|0]; game.coolUntil=now()+3000;}
    if(now()>game.coolUntil){
      const a=P(game.it);
      for(const id of ids){ if(id===game.it) continue; const b=P(id);
        if(Math.hypot(a.x-b.x,a.z-b.z)<1.1&&Math.abs(a.y-b.y)<1.4){game.it=id; game.coolUntil=now()+2500; k=id; break;} }
    }
    for(const id of ids) if(id!==game.it) game.s[id]=(game.s[id]||0)+dt;
  }
  const s={}; for(const id in game.s) s[id]=Math.floor(game.s[id]);
  const pk={t:'g',it:game.it,c:Math.max(0,game.coolUntil-now())|0,s,k}; sendAll(JSON.stringify(pk));
  game.s={...game.s}; if(k) beep(k===myId?200:700);
}
function startPeer(h){
  if(!window.Peer){setStatus('Could not load the multiplayer library. Check internet and refresh.');return;}
  if(peer) try{peer.destroy();}catch(e){}
  conns.clear(); clearOthers(); isHost=h; room=cleanRoom(roomInput.value)||'BANANA'; roomInput.value=room; $('#roomTag').textContent=room;
  const pid='monkeytag-'+room.toLowerCase();
  setStatus(h?'Creating room…':'Connecting to room…');
  peer=h?new Peer(pid):new Peer();
  peer.on('open',id=>{
    myId=id;
    if(h) setStatus('Room created! Share code '+room+' or use Copy link. Keep this tab open.');
    else{const c=peer.connect(pid,{reliable:true}); attach(c); setTimeout(()=>{if(!c.open)setStatus('Still connecting… check the code and that the host is online.');},6000);}
  });
  peer.on('connection',c=>{if(isHost)attach(c);});
  peer.on('error',e=>setStatus(e.type==='unavailable-id'?'That room code is taken. Choose another.':e.type==='peer-unavailable'?'Room not found. Make sure the host is online.':'Network error: '+e.type));
}
$('#host').addEventListener('click',()=>startPeer(true));
$('#join').addEventListener('click',()=>startPeer(false));
$('#vr').addEventListener('click',()=>{$('#help').hidden=!$('#help').hidden;});
$('#menu h1').addEventListener('click',()=>$('#menu').classList.toggle('min'));
$('#copy').addEventListener('click',()=>{
  const u=location.origin+location.pathname+'?room='+(cleanRoom(roomInput.value)||'BANANA');
  navigator.clipboard?.writeText(u).then(()=>setStatus('Invite link copied!'),()=>setStatus(u));
});
const qr=new URLSearchParams(location.search).get('room');
if(qr){roomInput.value=cleanRoom(qr); setTimeout(()=>startPeer(false),400);}

// ---------- Input ----------
const keys={}; let pointerLocked=false, yaw=0, pitch=0, heading=0;
renderer.domElement.addEventListener('click',()=>{if(matchMedia('(pointer:fine)').matches) renderer.domElement.requestPointerLock?.();});
document.addEventListener('pointerlockchange',()=>pointerLocked=document.pointerLockElement===renderer.domElement);
document.addEventListener('mousemove',e=>{if(pointerLocked){yaw-=e.movementX*.0024; pitch=clamp(pitch-e.movementY*.002,-1.15,1.15);}});
document.addEventListener('keydown',e=>{
  if(e.target.tagName==='INPUT') return;
  keys[e.code]=true; if(['Space','ArrowUp','ArrowDown'].includes(e.code)) e.preventDefault();
  if(e.code==='Space') jump();
});
document.addEventListener('keyup',e=>keys[e.code]=false);
$('#jump').addEventListener('pointerdown',e=>{e.preventDefault(); jump();});
const pad=$('#pad'), nub=$('#nub'); let joy={x:0,y:0}, padPointer=null;
function setJoy(e){
  const r=pad.getBoundingClientRect(); let dx=e.clientX-(r.left+r.width/2), dy=e.clientY-(r.top+r.height/2); const max=36,len=Math.hypot(dx,dy);
  if(len>max){dx=dx/len*max; dy=dy/len*max;} joy={x:dx/max,y:dy/max}; nub.style.transform=`translate(${dx}px,${dy}px)`;
}
function resetJoy(){padPointer=null; joy={x:0,y:0}; nub.style.transform='translate(0,0)';}
pad.addEventListener('pointerdown',e=>{padPointer=e.pointerId; pad.setPointerCapture(e.pointerId); setJoy(e);});
pad.addEventListener('pointermove',e=>{if(e.pointerId===padPointer)setJoy(e);});
pad.addEventListener('pointerup',resetJoy); pad.addEventListener('pointercancel',resetJoy);
// touch drag to look
let lookId=null,lx=0,ly=0;
renderer.domElement.addEventListener('pointerdown',e=>{if(e.pointerType==='touch'&&lookId===null){lookId=e.pointerId; lx=e.clientX; ly=e.clientY;}});
addEventListener('pointermove',e=>{if(e.pointerId===lookId){yaw-=(e.clientX-lx)*.005; pitch=clamp(pitch-(e.clientY-ly)*.004,-1.15,1.15); lx=e.clientX; ly=e.clientY;}});
const endLook=e=>{if(e.pointerId===lookId)lookId=null;}; addEventListener('pointerup',endLook); addEventListener('pointercancel',endLook);

// ---------- Player physics ----------
const pos=new THREE.Vector3(); {const a=Math.random()*6.283; pos.set(Math.cos(a)*6,0,Math.sin(a)*6); yaw=Math.atan2(pos.x,pos.z);}
let vy=0, grounded=true, snapped=false, jumpHeld=false, netT=0;
function jump(){if(grounded){vy=8.3; grounded=false;}}
function floorAt(x,z,py){
  let y=0;
  for(const p of platforms) if(Math.abs(x-p.x)<p.w/2+.2&&Math.abs(z-p.z)<p.d/2+.2&&py>=p.top-.05) y=Math.max(y,p.top);
  return y;
}
function move(dt,f,s,speed){
  const len=Math.hypot(f,s); if(len>1){f/=len; s/=len;}
  pos.x+=(-Math.sin(heading)*f+Math.cos(heading)*s)*speed*dt;
  pos.z+=(-Math.cos(heading)*f-Math.sin(heading)*s)*speed*dt;
  const r=Math.hypot(pos.x,pos.z); if(r>22.5){pos.x*=22.5/r; pos.z*=22.5/r;}
  const prev=pos.y; vy-=17*dt; pos.y+=vy*dt;
  const fy=floorAt(pos.x,pos.z,prev);
  if(pos.y<=fy){pos.y=fy; vy=0; grounded=true;} else grounded=false;
}

// ---------- Main loop ----------
const clock=new THREE.Clock(), tmp=new THREE.Vector3();
function animate(){
  const dt=Math.min(clock.getDelta(),.05), t=clock.elapsedTime, xr=renderer.xr.isPresenting;
  let f=0,s=0,speed=5;
  if(xr){
    camera.getWorldDirection(tmp); heading=Math.atan2(-tmp.x,-tmp.z); speed=4.5;
    let jb=false;
    for(const src of renderer.xr.getSession().inputSources){
      const gp=src.gamepad; if(!gp) continue; const a=gp.axes, ax=a.length>2?a[2]:a[0]||0, ay=a.length>3?a[3]:a[1]||0;
      if(src.handedness==='left'){s+=ax; f-=ay;}
      else if(src.handedness==='right'){
        if(Math.abs(ax)>.6){if(!snapped){yaw-=Math.sign(ax)*Math.PI/6; snapped=true;}} else snapped=false;
        jb=jb||!!gp.buttons[0]?.pressed||!!gp.buttons[4]?.pressed;
      }
    }
    if(jb&&!jumpHeld) jump(); jumpHeld=jb;
  }else{
    heading=yaw; speed=keys.ShiftLeft?8:5;
    if(keys.KeyW||keys.ArrowUp)f++; if(keys.KeyS||keys.ArrowDown)f--; if(keys.KeyD||keys.ArrowRight)s++; if(keys.KeyA||keys.ArrowLeft)s--;
    if(Math.abs(joy.x)+Math.abs(joy.y)>.08){s+=joy.x; f-=joy.y;}
  }
  if(game.it===myId) speed*=1.12;
  move(dt,f,s,speed);
  rig.position.copy(pos); rig.rotation.y=yaw;
  if(!xr){camera.position.set(0,1.65,0); camera.rotation.set(pitch,0,0);}
  hudS.visible=xr;

  const k=1-Math.exp(-12*dt), tNow=now();
  for(const [id,o] of others){
    if(tNow-o.seen>5000){removeOther(id); continue;}
    const d=o.g.position.distanceTo(o.tp), amp=Math.min(1,d*4);
    o.g.position.lerp(o.tp,k);
    let dy=o.ty-o.g.rotation.y; dy=Math.atan2(Math.sin(dy),Math.cos(dy)); o.g.rotation.y+=dy*k;
    const {arms,legs,mark}=o.g.userData, w=Math.sin(t*11)*amp*.9;
    arms[0].rotation.x=w; arms[1].rotation.x=-w; legs[0].rotation.x=-w; legs[1].rotation.x=w;
    mark.visible=game.it===id; mark.rotation.y=t*3; mark.position.y=2.65+Math.sin(t*4)*.08;
  }
  for(const c of clouds){c.position.x+=dt*.8; if(c.position.x>70)c.position.x=-70;}
  hoop.rotation.z=t*.8;

  netT+=dt;
  if(netT>=1/15){
    const step=netT; netT=0;
    if(conns.size){sendAll(JSON.stringify({t:'s',id:myId,p:[r2(pos.x),r2(pos.y),r2(pos.z),r2(xr?heading:yaw)],n:myName(),c:'#'+myColor.getHexString()}));}
    if(isHost) hostTick(step);
    refreshHud();
  }
  renderer.render(scene,camera);
}
renderer.setAnimationLoop(animate);
refreshHud();
addEventListener('resize',()=>{camera.aspect=innerWidth/innerHeight; camera.updateProjectionMatrix(); renderer.setSize(innerWidth,innerHeight);});
</script>
</body>
</html>
