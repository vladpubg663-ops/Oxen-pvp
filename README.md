# Oxen-pvp
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=5,maximum-scale=1,user-scalable=no">
<title>OXEN PVP</title>
<style>
*{box-sizing:border-box}
body{margin:0;background:#171b20;color:white;font-family:Arial;text-align:center}
h1{color:#50dc8b}
button,input{padding:13px;margin:5px;border:0;border-radius:9px;font-size:16px}
button{background:#36c978;color:#07140c;font-weight:bold}
input{width:150px}
#game{display:none}
canvas{background:#293b30;width:100%;max-width:900px;touch-action:none}
#controls button{width:65px;height:55px;background:#444;color:white}
</style>
</head>
<body>
<h1>OXEN PVP</h1>
<div id="menu">
<p>PVP — до 5 игроков</p>
<button onclick="host()">Создать комнату</button>
<br>
<input id="room" placeholder="Код комнаты">
<button onclick="join()">Войти</button>
<p id="status">Не подключено</p>
</div>
<div id="game">
<p>HP: <span id="hp">100</span> | Игроков: <span id="count">1</span></p>
<canvas id="c" width="900" height="550"></canvas>
<div id="controls">
<button ontouchstart="keys.w=1" ontouchend="keys.w=0">↑</button><br>
<button ontouchstart="keys.a=1" ontouchend="keys.a=0">←</button>
<button ontouchstart="keys.s=1" ontouchend="keys.s=0">↓</button>
<button ontouchstart="keys.d=1" ontouchend="keys.d=0">→</button>
<button ontouchstart="fire()" onclick="fire()">ОГОНЬ</button>
</div>
</div>
<script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
<script>
const c=document.getElementById('c'),ctx=c.getContext('2d');
let peer,hostPlayer=false,connections=[],players={},bullets=[];
let myId='',roomCode='',hp=100,lastShot=0;
const keys={w:0,a:0,s:0,d:0};
let me={x:450,y:275,color:'#42e879'};
function say(t){document.getElementById('status').textContent=t}
function start(){
 document.getElementById('menu').style.display='none';
 document.getElementById('game').style.display='block';
 players[myId]=me;
 requestAnimationFrame(loop);
}
function host(){
 roomCode=Math.random().toString(36).slice(2,8);
 peer=new Peer('oxen-'+roomCode);
 peer.on('open',id=>{
  myId=id;hostPlayer=true;
  document.getElementById('room').value=roomCode;
  say('Комната: '+roomCode);
  start();
 });
 peer.on('connection',conn=>{
  if(connections.length>=4){conn.close();return}
  connections.push(conn);
  conn.on('data',msg=>{
   if(msg.type==='move'){
    players[conn.peer]={x:msg.x,y:msg.y,color:'#ffbd4a'};
    broadcast({type:'state',players});
   }
   if(msg.type==='shoot'){
    bullets.push({x:msg.x,y:msg.y,dx:msg.dx,dy:msg.dy,owner:conn.peer});
   }
  });
  conn.on('close',()=>{
   connections=connections.filter(x=>x!==conn);
   delete players[conn.peer];
  });
  conn.on('open',()=>{
   conn.send({type:'welcome',id:conn.peer,players});
  });
 });
}
function join(){
 roomCode=document.getElementById('room').value.trim();
 if(!roomCode)return say('Введи код комнаты');
 peer=new Peer();
 peer.on('open',id=>{
  myId=id;
  let conn=peer.connect('oxen-'+roomCode);
  conn.on('open',()=>{
   hostConn=conn;
   conn.send({type:'join'});
   conn.on('data',msg=>{
    if(msg.type==='welcome'){players=msg.players||{};start()}
    if(msg.type==='state')players=msg.players;
    if(msg.type==='hit'&&msg.id===myId){
     hp=Math.max(0,hp-20);
     document.getElementById('hp').textContent=hp;
     if(hp===0){hp=100;me.x=450;me.y=275}
    }
   });
   say('Подключено');
  });
  conn.on('error',()=>say('Не удалось войти. Проверь код.'));
 });
}
let hostConn=null;
function broadcast(data){
 connections.forEach(conn=>{if(conn.open)conn.send(data)});
}
function fire(){
 if(Date.now()-lastShot<350)return;
 lastShot=Date.now();
 let dx=1,dy=0;
 let b={x:me.x,y:me.y,dx,dy,owner:myId};
 if(hostPlayer)bullets.push(b);
 else if(hostConn)hostConn.send({type:'shoot',...b});
}
document.addEventListener('keydown',e=>{
 let k=e.key.toLowerCase();if(k in keys)keys[k]=1;
 if(k===' ')fire();
});
document.addEventListener('keyup',e=>{
 let k=e.key.toLowerCase();if(k in keys)keys[k]=0;
});
function loop(){
 if(keys.w)me.y-=3;
 if(keys.s)me.y+=3;
 if(keys.a)me.x-=3;
 if(keys.d)me.x+=3;
 me.x=Math.max(15,Math.min(885,me.x));
 me.y=Math.max(15,Math.min(535,me.y));
 if(hostPlayer){
  players[myId]=me;
  broadcast({type:'state',players});
 }else if(hostConn){
  hostConn.send({type:'move',x:me.x,y:me.y});
 }
 ctx.clearRect(0,0,900,550);
 ctx.fillStyle='#314b39';
 ctx.fillRect(0,0,900,550);
 for(let id in players){
  let p=players[id];
  ctx.fillStyle=id===myId?'#42e879':'#ffbd4a';
  ctx.beginPath();ctx.arc(p.x,p.y,15,0,Math.PI*2);ctx.fill();
  ctx.fillStyle='white';ctx.fillText(id===myId?'ТЫ':'ИГРОК',p.x-20,p.y-23);
 }
 if(hostPlayer){
  bullets.forEach(b=>{
   b.x+=b.dx*8;b.y+=b.dy*8;
   for(let id in players){
    if(id!==b.owner){
     let p=players[id];
     if(Math.hypot(p.x-b.x,p.y-b.y)<20){
      if(id===myId){hp=Math.max(0,hp-20);document.getElementById('hp').textContent=hp}
      else{
       let conn=connections.find(x=>x.peer===id);
       if(conn&&conn.open)conn.send({type:'hit',id});
      }
      b.dead=true;
     }
    }
   }
  });
  bullets=bullets.filter(b=>!b.dead&&b.x>0&&b.x<900&&b.y>0&&b.y<550);
 }
 bullets.forEach(b=>{
  ctx.fillStyle='#fff';ctx.beginPath();ctx.arc(b.x,b.y,5,0,7);ctx.fill();
 });
 document.getElementById('count').textContent=Object.keys(players).length;
 requestAnimationFrame(loop);
}
</script>
</body>
</html>
