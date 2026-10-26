[index (1).html](https://github.com/user-attachments/files/33952099/index.1.html)<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>Monkey Tag — VR Multiplayer</title>
<style>
  :root { color-scheme: dark; --panel:rgba(10,18,28,.88); --accent:#80ffb0; }
  * { box-sizing:border-box; }
  html,body { margin:0; width:100%; height:100%; overflow:hidden; font-family:system-ui,Arial,sans-serif; background:#87ceeb; }
  canvas { display:block; }
  #menu { position:fixed; z-index:5; left:12px; top:12px; width:min(350px,calc(100vw - 24px)); padding:14px; border:1px solid #ffffff44; border-radius:16px; background:var(--panel); color:white; box-shadow:0 8px 30px #0005; }
  h1 { margin:0 0 5px; font-size:22px; } p { margin:6px 0 10px; font-size:13px; line-height:1.35; color:#d5e4ef; }
  .line { display:flex; gap:7px; margin:8px 0; }
  input,button { min-width:0; border:0; border-radius:10px; padding:11px 12px; font:inherit; }
  input { flex:1; background:#f5f8fc; color:#15202b; text-transform:uppercase; }
  button { cursor:pointer; font-weight:750; background:var(--accent); color:#10251a; }
  button.secondary { background:#314454; color:white; }
  #status { font-size:12px; color:#bcebd0; min-height:17px; overflow-wrap:anywhere; }
  #help { position:fixed; z-index:3; right:12px; top:12px; max-width:260px; padding:10px 12px; border-radius:12px; background:#0a121cc9; color:white; font-size:12px; line-height:1.4; }
  #roomTag { font-weight:800; color:var(--accent); }
  #mobile { display:none; position:fixed; z-index:4; left:0; right:0; bottom:14px; padding:0 18px; justify-content:space-between; align-items:end; pointer-events:none; }
  .pad { width:118px; height:118px; border-radius:50%; background:#14233277; border:2px solid #ffffff88; position:relative; pointer-events:auto; touch-action:none; }
  .nub { width:46px; height:46px; border-radius:50%; position:absolute; left:34px; top:34px; background:#e9fff0dd; border:2px solid #fff; }
  #jump { width:76px; height:76px; border-radius:50%; pointer-events:auto; touch-action:none; background:#80ffb0dd; border:2px solid white; }
  #vrNotice { position:fixed; z-index:2; bottom:12px; left:50%; transform:translateX(-50%); padding:7px 10px; border-radius:9px; color:white; background:#0a121c99; font-size:12px; white-space:nowrap; }
  @media (pointer:coarse) { #mobile { display:flex; } #help { top:auto; bottom:145px; right:10px; max-width:175px; } #menu { width:min(340px,calc(100vw - 24px)); } }
</style>
</head>
<body>
<div id="menu">
  <h1>🐒 Monkey Tag</h1>
  <p>One shared 3D playground. Create a room, then share its code. The room creator must keep the game open.</p>
  <div class="line"><input id="room" maxlength="12" placeholder="ROOM CODE" value="BANANA"><button id="host">Create room</button></div>
  <div class="line"><button id="join" class="secondary" style="flex:1">Join room</button><button id="vr" class="secondary">VR help</button></div>
  <div id="status">Not connected. Choose Create room or Join room.</div>
  <p>Room: <span id="roomTag">—</span></p>
</div>
<div id="help">🖥️ Laptop: WASD to move, mouse to look (click game first), Space to jump.<br>📱 Phone: use the left pad and JUMP.<br>🥽 Quest: press Enter VR; use left thumbstick to move and right thumbstick to turn.</div>
<div id="mobile"><div class="pad" id="pad"><div class="nub" id="nub"></div></div><button id="jump">JUMP</button></div>
<div id="vrNotice">Walk, jump, and tag your friends!</div>

<script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
<script type="importmap">
{"imports":{"three":"https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js","three/addons/":"https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"}}
</script>
<script type="module">
import * as THREE from 'three';
import { VRButton } from 'three/addons/webxr/VRButton.js';

const statusEl = document.querySelector('#status');
const roomInput = document.querySelector('#room');
const roomTag = document.querySelector('#roomTag');
const setStatus = s => statusEl.textContent = s;
const cleanRoom = s => s.toUpperCase().replace(/[^A-Z0-9]/g,'').slice(0,12);
const myId = 'p' + Math.random().toString(36).slice(2,10);
let peer = null, isHost = false, currentRoom = '', connections = new Map();
let myColor = new THREE.Color().setHSL(Math.random(), .78, .55);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x8bd7ff);
scene.fog = new THREE.Fog(0x8bd7ff, 28, 90);
const camera = new THREE.PerspectiveCamera(70, innerWidth/innerHeight, .1, 150);
const playerRig = new THREE.Group();
scene.add(playerRig);
playerRig.add(camera);
camera.position.set(0,1.65,0);
const renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setPixelRatio(Math.min(devicePixelRatio,1.7));
renderer.setSize(innerWidth,innerHeight);
renderer.xr.enabled = true;
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);
const vrButton = VRButton.createButton(renderer);
vrButton.style.position='fixed'; vrButton.style.bottom='12px'; vrButton.style.left='50%'; vrButton.style.transform='translateX(-50%)'; vrButton.style.zIndex='4';
document.body.appendChild(vrButton);

scene.add(new THREE.HemisphereLight(0xdff6ff,0x4c7547,2));
const sun = new THREE.DirectionalLight(0xffffff,2.2); sun.position.set(-8,15,6); sun.castShadow=true; scene.add(sun);

const mat = (color,roughness=.85)=>new THREE.MeshStandardMaterial({color,roughness});
const grassMat=mat(0x45b94b), dirtMat=mat(0x80512e), trunkMat=mat(0x79502d), leafMat=mat(0x168d55);
const ground = new THREE.Mesh(new THREE.CylinderGeometry(24,25,1.2,48), grassMat);
ground.position.y=-.65; ground.receiveShadow=true; scene.add(ground);
const dirt = new THREE.Mesh(new THREE.CylinderGeometry(24,25,1.1,48),dirtMat); dirt.position.y=-1.28; scene.add(dirt);
const grid = new THREE.GridHelper(48,48,0xffffff,0xffffff); grid.position.y=-.04; grid.material.transparent=true; grid.material.opacity=.12; scene.add(grid);

function addTree(x,z,s=1){
  const g=new THREE.Group(); g.position.set(x,0,z); g.scale.setScalar(s);
  const trunk=new THREE.Mesh(new THREE.CylinderGeometry(.18,.28,2,7),trunkMat); trunk.position.y=1; trunk.castShadow=true; g.add(trunk);
  for(let i=0;i<3;i++){ const crown=new THREE.Mesh(new THREE.IcosahedronGeometry(1.05-i*.12,1),leafMat); crown.position.set((i-1)*.18,2.1+i*.48,0); crown.castShadow=true; g.add(crown); }
  scene.add(g);
}
for(const [x,z,s] of [[-8,-7,1.3],[-12,2,1.1],[9,-9,1.2],[13,1,1.1],[5,13,1.4],[-7,12,1.1],[16,8,.9],[-17,-5,.9]]) addTree(x,z,s);
for(let i=0;i<12;i++){
  const rock=new THREE.Mesh(new THREE.DodecahedronGeometry(.35+Math.random()*.45,0),mat(0x8996a0));
  const a=Math.random()*Math.PI*2, r=5+Math.random()*16; rock.position.set(Math.cos(a)*r,.05,Math.sin(a)*r); rock.rotation.set(Math.random(),Math.random(),Math.random()); rock.castShadow=true; scene.add(rock);
}
const platforms=[];
for(const [x,y,z,w,d] of [[-4,1.4,-3,3,2],[1,2.5,-7,3,2],[6,1.4,-4,3,2],[-7,2,4,3,2],[4,3,5,3,2]]){
 const m=new THREE.Mesh(new THREE.BoxGeometry(w,.35,d),mat(0xffc857)); m.position.set(x,y,z); m.castShadow=true; m.receiveShadow=true; scene.add(m); platforms.push({x,y,z,w,d});
}
const hoop=new THREE.Mesh(new THREE.TorusGeometry(1.2,.12,10,28),mat(0xff5e75)); hoop.position.set(0,2.3,-10); hoop.rotation.x=Math.PI/2; scene.add(hoop);

// Local avatar and player state
const avatar = new THREE.Group(); scene.add(avatar);
const body = new THREE.Mesh(new THREE.CapsuleGeometry(.38,.65,4,8),new THREE.MeshStandardMaterial({color:myColor})); body.position.y=.75; body.castShadow=true; avatar.add(body);
const head = new THREE.Mesh(new THREE.SphereGeometry(.34,16,12),new THREE.MeshStandardMaterial({color:0xffd6aa})); head.position.y=1.45; avatar.add(head);
const eyeMat=mat(0x17202b);
for(const x of [-.12,.12]){const eye=new THREE.Mesh(new THREE.SphereGeometry(.045,8,8),eyeMat);eye.position.set(x,1.49,-.29);avatar.add(eye);}
const nameCanvas=document.createElement('canvas'); nameCanvas.width=256; nameCanvas.height=64;
const nameCtx=nameCanvas.getContext('2d'); nameCtx.fillStyle='white'; nameCtx.font='bold 28px sans-serif'; nameCtx.textAlign='center'; nameCtx.fillText('YOU',128,40);
const nameTex=new THREE.CanvasTexture(nameCanvas);
const nameSprite=new THREE.Sprite(new THREE.SpriteMaterial({map:nameTex,transparent:true})); nameSprite.scale.set(1.5,.38,1); nameSprite.position.y=2.05; avatar.add(nameSprite);

let pos=new THREE.Vector3(0,0,4), velocityY=0, grounded=true, yaw=0, pitch=0;
let lastNetworkSend = 0;
const others=new Map();
function makeOther(id, data){
 const g=new THREE.Group();
 const c=new THREE.Color(data?.color ?? '#ff8a40');
 const b=new THREE.Mesh(new THREE.CapsuleGeometry(.38,.65,4,8),new THREE.MeshStandardMaterial({color:c})); b.position.y=.75; g.add(b);
 const h=new THREE.Mesh(new THREE.SphereGeometry(.34,12,10),mat(0xffd6aa)); h.position.y=1.45; g.add(h);
 const labelCanvas=document.createElement('canvas'); labelCanvas.width=256; labelCanvas.height=64;
 const cx=labelCanvas.getContext('2d'); cx.fillStyle='white'; cx.font='bold 25px sans-serif'; cx.textAlign='center'; cx.fillText(id.slice(-5).toUpperCase(),128,40);
 const tex=new THREE.CanvasTexture(labelCanvas); const sp=new THREE.Sprite(new THREE.SpriteMaterial({map:tex,transparent:true})); sp.scale.set(1.6,.4,1); sp.position.y=2.05; g.add(sp);
 scene.add(g); return g;
}
function sendAll(packet){
 const msg=JSON.stringify(packet);
 for(const [id,c] of connections){ if(c.open){ try{c.send(msg);}catch(e){} } }
}
function sendState(){
 const p={t:'state',id:myId,x:pos.x,y:pos.y,z:pos.z,yaw,color:'#'+myColor.getHexString()};
 sendAll(p);
}
function onPacket(raw, c){
 let p; try{p=typeof raw==='string'?JSON.parse(raw):raw;}catch(e){return;}
 if(p.t==='hello'){
   if(isHost){ sendAll({t:'welcome',id:myId,players:[...others.entries()].map(([id,g])=>({id,x:g.position.x,y:g.position.y,z:g.position.z,color:'#ff8a40'}))}); }
   c.send(JSON.stringify({t:'welcome',id:myId,players:[...others.keys()].map(id=>({id,x:others.get(id).position.x,y:others.get(id).position.y,z:others.get(id).position.z,color:'#ff8a40'}))}));
   return;
 }
 if(p.t==='state' && p.id!==myId){
   if(!others.has(p.id)) others.set(p.id,makeOther(p.id,p));
   const g=others.get(p.id); g.position.set(Number(p.x)||0,Number(p.y)||0,Number(p.z)||0); g.rotation.y=Number(p.yaw)||0;
   if(p.color){g.children[0].material.color.set(p.color);}
   // relay updates through host to other room members
   if(isHost){for(const [id,conn] of connections){if(conn!==c&&conn.open) try{conn.send(JSON.stringify(p));}catch(e){}}}
 } else if(p.t==='welcome'){
   (p.players||[]).forEach(o=>{if(o.id!==myId&&!others.has(o.id)){const g=makeOther(o.id,o);g.position.set(o.x||0,o.y||0,o.z||0);others.set(o.id,g);}});
 }
}
function attachConnection(c){
 connections.set(c.peer,c);
 c.on('open',()=>{ c.send(JSON.stringify({t:'hello',id:myId,color:'#'+myColor.getHexString()})); setStatus('Connected! Players can now explore together.'); });
 c.on('data',raw=>onPacket(raw,c));
 c.on('close',()=>{connections.delete(c.peer); if(others.has(c.peer)){scene.remove(others.get(c.peer));others.delete(c.peer);} });
 c.on('error',()=>setStatus('Connection issue. Try reconnecting or a shorter room code.'));
}
function startPeer(hostMode){
 if(!window.Peer){setStatus('Could not load the multiplayer library. Check internet and refresh.');return;}
 if(peer){try{peer.destroy();}catch(e){}}
 connections.clear(); isHost=hostMode; currentRoom=cleanRoom(roomInput.value)||'BANANA'; roomInput.value=currentRoom; roomTag.textContent=currentRoom;
 const peerId='monkeytag-'+currentRoom.toLowerCase();
 setStatus(hostMode?'Creating room…':'Connecting to room…');
 // Room codes are public identifiers, not passwords. Anyone who knows one may join.
 peer = hostMode ? new Peer(peerId) : new Peer();
 peer.on('open', id=>{
   if(hostMode){setStatus('Room created! Share code '+currentRoom+'. Keep this tab open.');}
   else {
     const c=peer.connect(peerId,{reliable:true}); attachConnection(c);
     setTimeout(()=>{if(!c.open)setStatus('Still connecting… check the code and ensure the room creator is online.');},5000);
   }
 });
 peer.on('connection', c=>{if(isHost)attachConnection(c);});
 peer.on('error',err=>{
   if(err.type==='unavailable-id')setStatus('That room code is already in use. Choose another code.');
   else if(err.type==='peer-unavailable')setStatus('Room not found. Check the code and make sure the creator is online.');
   else setStatus('Network error: '+err.type+'. Try again.');
 });
}
document.querySelector('#host').addEventListener('click',()=>startPeer(true));
document.querySelector('#join').addEventListener('click',()=>startPeer(false));
document.querySelector('#vr').addEventListener('click',()=>{document.querySelector('#help').hidden=!document.querySelector('#help').hidden;});

// Desktop mouse-look and keyboard
const keys={}; let pointerLocked=false;
renderer.domElement.addEventListener('click',()=>{if(matchMedia('(pointer:fine)').matches) renderer.domElement.requestPointerLock?.();});
document.addEventListener('pointerlockchange',()=>pointerLocked=document.pointerLockElement===renderer.domElement);
document.addEventListener('mousemove',e=>{if(pointerLocked){yaw-=e.movementX*.0024;pitch-=e.movementY*.002;pitch=Math.max(-1.15,Math.min(1.15,pitch));}});
document.addEventListener('keydown',e=>{keys[e.code]=true;if(['Space','ArrowUp','ArrowDown'].includes(e.code))e.preventDefault();});
document.addEventListener('keyup',e=>keys[e.code]=false);
document.querySelector('#jump').addEventListener('pointerdown',e=>{e.preventDefault();if(grounded){velocityY=7;grounded=false;}});
document.addEventListener('keydown',e=>{if(e.code==='Space'&&grounded){velocityY=7;grounded=false;}});

// Mobile virtual joystick
const pad=document.querySelector('#pad'), nub=document.querySelector('#nub');
let joy={x:0,y:0},padPointer=null;
function setJoy(e){
 const r=pad.getBoundingClientRect(),cx=r.left+r.width/2,cy=r.top+r.height/2;
 let dx=e.clientX-cx,dy=e.clientY-cy;const max=36, len=Math.hypot(dx,dy);
 if(len>max){dx=dx/len*max;dy=dy/len*max;}
 joy={x:dx/max,y:dy/max};nub.style.transform=`translate(${dx}px,${dy}px)`;
}
pad.addEventListener('pointerdown',e=>{padPointer=e.pointerId;pad.setPointerCapture(e.pointerId);setJoy(e);});
pad.addEventListener('pointermove',e=>{if(e.pointerId===padPointer)setJoy(e);});
function resetJoy(){padPointer=null;joy={x:0,y:0};nub.style.transform='translate(0,0)';}
pad.addEventListener('pointerup',resetJoy);pad.addEventListener('pointercancel',resetJoy);

const clock=new THREE.Clock();
const tempDir=new THREE.Vector3();
function onFloor(x,z){
 let y=0;
 for(const p of platforms){if(Math.abs(x-p.x)<p.w/2+.25&&Math.abs(z-p.z)<p.d/2+.25&&pos.y<=p.y+.3&&pos.y>=p.y-.6)y=Math.max(y,p.y+.2);}
 return y;
}
function updateDesktop(dt){
 let forward=0,side=0;
 if(keys.KeyW||keys.ArrowUp)forward+=1;if(keys.KeyS||keys.ArrowDown)forward-=1;
 if(keys.KeyD||keys.ArrowRight)side+=1;if(keys.KeyA||keys.ArrowLeft)side-=1;
 if(Math.abs(joy.x)+Math.abs(joy.y)>.08){side+=joy.x;forward-=joy.y;}
 const len=Math.hypot(forward,side);if(len>1){forward/=len;side/=len;}
 const speed=keys.ShiftLeft?8:5;
 const dx=(Math.sin(yaw)*forward+Math.cos(yaw)*side)*speed*dt;
 const dz=(-Math.cos(yaw)*forward+Math.sin(yaw)*side)*speed*dt;
 pos.x+=dx;pos.z+=dz;
 pos.x=THREE.MathUtils.clamp(pos.x,-22,22);pos.z=THREE.MathUtils.clamp(pos.z,-22,22);
 velocityY-=18*dt;pos.y+=velocityY*dt;
 const floorY=onFloor(pos.x,pos.z);
 if(pos.y<=floorY){pos.y=floorY;velocityY=0;grounded=true;}
 if(pos.y<0){pos.y=0;velocityY=0;grounded=true;}
}
function updateVR(dt){
 const session=renderer.xr.getSession();
 if(!session){updateDesktop(dt);return;}
 const controllers=[renderer.xr.getController(0),renderer.xr.getController(1)];
 let forward=0,side=0,turn=0;
 for(let i=0;i<controllers.length;i++){
   const src=session.inputSources?.[i],gp=src?.gamepad;if(!gp)continue;
   const ax=gp.axes||[];
   if(i===0){side+=ax[2]??ax[0]??0;forward-=ax[3]??ax[1]??0;}
   if(i===1)turn+=ax[2]??ax[0]??0;
 }
 if(Math.abs(turn)>.25)yaw-=turn*dt*2.1;
 const len=Math.hypot(forward,side);if(len>1){forward/=len;side/=len;}
 pos.x+=(Math.sin(yaw)*forward+Math.cos(yaw)*side)*4.5*dt;
 pos.z+=(-Math.cos(yaw)*forward+Math.sin(yaw)*side)*4.5*dt;
 pos.x=THREE.MathUtils.clamp(pos.x,-22,22);pos.z=THREE.MathUtils.clamp(pos.z,-22,22);
 velocityY-=18*dt;pos.y+=velocityY*dt;
 const fy=onFloor(pos.x,pos.z);
 if(pos.y<=fy){pos.y=fy;velocityY=0;grounded=true;}
 if(pos.y<0){pos.y=0;velocityY=0;grounded=true;}
}
function animate(){
 const dt=Math.min(clock.getDelta(),.04);
 updateVR(dt);
 avatar.position.copy(pos);avatar.rotation.y=yaw;
 if(!renderer.xr.isPresenting){
   camera.position.set(pos.x,pos.y+1.65,pos.z);
   camera.rotation.order='YXZ';camera.rotation.y=yaw;camera.rotation.x=pitch;
 }else{
   // Keep the player's real headset view; locomotion offsets the camera rig through the avatar position.
   const rig=renderer.xr.getCamera();
   // VR camera is driven by the headset; offset its parented rig via scene-level avatar position is not possible,
   // so translate the XR camera's parent rig using a dedicated origin group below.
 }
 sendState();
 renderer.render(scene,camera);
}
renderer.setAnimationLoop(animate);
window.addEventListener('resize',()=>{camera.aspect=innerWidth/innerHeight;camera.updateProjectionMatrix();renderer.setSize(innerWidth,innerHeight);});
</script>
</body>
</html>
