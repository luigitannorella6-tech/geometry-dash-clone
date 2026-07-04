<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><meta name="viewport" content="width=device-width,initial-scale=1.0"><title>HTML5 Platformer Engine - Stage 3</title>
<style>
body{margin:0;overflow:hidden;background-color:#1a1a1a;color:white;font-family:sans-serif;user-select:none;}
canvas{display:block;}
#ui-layer{position:absolute;top:10px;left:10px;z-index:10;pointer-events:none;display:none;}
.ui-btn{pointer-events:auto;background:#333;color:#fff;border:1px solid #555;padding:5px 10px;cursor:pointer;margin-right:5px;}
.ui-btn:hover{background:#555;}
.overlay{position:absolute;top:0;left:0;width:100vw;height:100vh;background:rgba(0,0,0,0.85);display:flex;flex-direction:column;justify-content:center;align-items:center;z-index:20;}
.overlay h1{font-size:3.5rem;margin-bottom:30px;color:#00ffaa;text-shadow:0 0 10px rgba(0,255,170,0.5);}
#game-over-menu h1{color:#ff3366;text-shadow:0 0 10px rgba(255,51,102,0.5);}
.menu-btn{background:#00ffaa;color:#1a1a1a;border:none;padding:15px 40px;font-size:1.5rem;margin:10px;cursor:pointer;border-radius:5px;font-weight:bold;text-transform:uppercase;transition:transform 0.1s,background 0.1s;}
.menu-btn:hover{background:#33ffbb;transform:scale(1.05);}
.menu-btn.danger{background:#ff3366;color:white;}
.menu-btn.danger:hover{background:#ff5588;}
.import-section{margin-top:40px;padding:20px;background:#222;border-radius:8px;display:flex;gap:10px;align-items:center;border:1px solid #444;}
.import-input{padding:10px;font-size:1rem;width:300px;border-radius:4px;border:1px solid #555;background:#111;color:#00ffaa;}
.import-input:focus{outline:none;border-color:#00ffaa;}
.import-btn{background:#444;color:white;border:none;padding:10px 20px;font-size:1rem;cursor:pointer;border-radius:4px;}
.import-btn:hover{background:#666;}
</style>
</head>
<body>
<div id="main-menu" class="overlay">
    <h1>GEOMETRY DASH CLONE</h1>
    <button class="menu-btn" onclick="window.gameEngine.changeState('PLAYING')">Play</button>
    <button class="menu-btn" onclick="window.gameEngine.changeState('EDITING')">Level Editor</button>
    <div class="import-section">
        <input type="text" id="menu-import-code" class="import-input" placeholder="Paste Level Code Here...">
        <button class="import-btn" onclick="window.gameEngine.importFromMenu()">Load & Play</button>
    </div>
</div>
<div id="game-over-menu" class="overlay" style="display:none;">
    <h1>GAME OVER</h1>
    <button class="menu-btn danger" onclick="window.gameEngine.changeState('PLAYING')">Retry</button>
    <button class="menu-btn" onclick="window.gameEngine.changeState('MENU')">Main Menu</button>
</div>
<div id="ui-layer">
    <button class="ui-btn" onclick="window.gameEngine.exportLevel()">Export Level</button>
    <button class="ui-btn" onclick="window.gameEngine.importLevel()">Import Level</button>
    <span id="mode-text">Mode: EDIT (Press 'E' to test) - Use Arrow/WASD Keys to pan</span>
</div>
<canvas id="gameCanvas"></canvas>
<script>
window.gameEngine={
    canvas:null,ctx:null,tileSize:40,cameraX:0,state:'MENU',levelData:{},keys:{},
    particles:[],shakeTimer:0,
    player:{x:120,y:150,size:30,vx:7,vy:0,gravity:0.65,jumpForce:-12.5,isGrounded:false},
    init(){
        this.canvas=document.getElementById('gameCanvas');this.ctx=this.canvas.getContext('2d');this.resize();
        window.addEventListener('resize',()=>this.resize());
        window.addEventListener('keydown',(e)=>this.handleKeyDown(e));
        window.addEventListener('keyup',(e)=>this.keys[e.code]=false);
        this.canvas.addEventListener('mousedown',(e)=>this.handleMouse(e));
        this.canvas.addEventListener('contextmenu',(e)=>e.preventDefault());
        this.generateDefaultFloor();
        requestAnimationFrame(()=>this.loop());
    },
    changeState(newState){
        this.state=newState;
        document.getElementById('main-menu').style.display=(newState==='MENU')?'flex':'none';
        document.getElementById('game-over-menu').style.display=(newState==='GAMEOVER')?'flex':'none';
        document.getElementById('ui-layer').style.display=(newState==='EDITING')?'block':'none';
        if(newState==='PLAYING'||newState==='EDITING')this.resetPlayer();
        if(newState==='GAMEOVER'){
            this.shakeTimer=15;
            for(let i=0;i<25;i++){
                this.particles.push({
                    x:this.player.x+this.player.size/2,y:this.player.y+this.player.size/2,
                    vx:(Math.random()-0.5)*15,vy:(Math.random()-0.5)*15,
                    life:1,color:'#ff3366'
                });
            }
        }
    },
    resize(){this.canvas.width=window.innerWidth;this.canvas.height=window.innerHeight;},
    generateDefaultFloor(){for(let i=0;i<25;i++)this.levelData[`${i},10`]=1;},
    handleKeyDown(e){
        this.keys[e.code]=true;
        if(e.code==='KeyE'){
            if(this.state==='PLAYING')this.changeState('EDITING');
            else if(this.state==='EDITING')this.changeState('PLAYING');
        }
    },
    handleMouse(e){
        if(this.state!=='EDITING')return;
        const rect=this.canvas.getBoundingClientRect();
        const gridX=Math.floor((e.clientX-rect.left+this.cameraX)/this.tileSize);
        const gridY=Math.floor((e.clientY-rect.top)/this.tileSize);
        if(e.button===0)this.levelData[`${gridX},${gridY}`]=1;
        else if(e.button===2)delete this.levelData[`${gridX},${gridY}`];
    },
    resetPlayer(){
        this.player.x=120;
        this.player.y=(10*this.tileSize)-this.player.size-4;
        this.player.vy=0;this.cameraX=0;this.player.isGrounded=false;
        this.keys['Space']=false;this.keys['ArrowUp']=false;this.keys['KeyW']=false;
        this.particles=[];
    },
    checkCollision(newX,newY){
        const p=this.player,ts=this.tileSize;
        const l=Math.floor(newX/ts),r=Math.floor((newX+p.size-0.01)/ts);
        const t=Math.floor(newY/ts),b=Math.floor((newY+p.size-0.01)/ts);
        for(let c=l;c<=r;c++){for(let r=t;r<=b;r++){if(this.levelData[`${c},${r}`])return true;}}
        return false;
    },
    update(){
        if(this.shakeTimer>0)this.shakeTimer--;
        for(let i=this.particles.length-1;i>=0;i--){
            let pt=this.particles[i];
            pt.x+=pt.vx;pt.y+=pt.vy;pt.life-=0.03;
            if(pt.life<=0)this.particles.splice(i,1);
        }
        if(this.state==='MENU'||this.state==='GAMEOVER')return;
        if(this.state==='EDITING'){
            if(this.keys['ArrowRight']||this.keys['KeyD'])this.cameraX+=12;
            if(this.keys['ArrowLeft']||this.keys['KeyA'])this.cameraX-=12;
            if(this.cameraX<0)this.cameraX=0;
            return;
        }
        const p=this.player;p.vy+=p.gravity;
        if(!this.checkCollision(p.x,p.y+p.vy)){
            p.y+=p.vy;p.isGrounded=false;
        }else{
            if(p.vy>0){p.isGrounded=true;p.y=Math.floor((p.y+p.vy)/this.tileSize)*this.tileSize-p.size;}
            else if(p.vy<0)p.y=Math.ceil(p.y/this.tileSize)*this.tileSize;
            p.vy=0;
        }
        if(p.isGrounded&&(this.keys['Space']||this.keys['ArrowUp']||this.keys['KeyW'])){p.vy=p.jumpForce;p.isGrounded=false;}
        
        if(!this.checkCollision(p.x+p.vx,p.y)){
            p.x+=p.vx;
            if(Math.random()>0.5){
                this.particles.push({
                    x:p.x,y:p.y+p.size-5+Math.random()*5,
                    vx:-p.vx*0.3,vy:(Math.random()-0.5)*2,
                    life:1,color:'#ff3366'
                });
            }
        }else{this.changeState('GAMEOVER');return;}
        if(p.y>this.canvas.height+150){this.changeState('GAMEOVER');return;}
        this.cameraX=p.x-200;if(this.cameraX<0)this.cameraX=0;
    },
    draw(){
        this.ctx.clearRect(0,0,this.canvas.width,this.canvas.height);
        this.ctx.save();
        let sx=0,sy=0;
        if(this.shakeTimer>0){sx=(Math.random()-0.5)*15;sy=(Math.random()-0.5)*15;}
        this.ctx.translate(-this.cameraX+sx,sy);
        if(this.state==='EDITING'){
            this.ctx.strokeStyle='#333';this.ctx.lineWidth=1;
            const startX=Math.floor(this.cameraX/this.tileSize)*this.tileSize;
            for(let x=startX;x<this.cameraX+this.canvas.width;x+=this.tileSize){this.ctx.beginPath();this.ctx.moveTo(x,0);this.ctx.lineTo(x,this.canvas.height);this.ctx.stroke();}
            for(let y=0;y<this.canvas.height;y+=this.tileSize){this.ctx.beginPath();this.ctx.moveTo(this.cameraX,y);this.ctx.lineTo(this.cameraX+this.canvas.width,y);this.ctx.stroke();}
        }
        
        this.ctx.shadowBlur=10;this.ctx.shadowColor='#00ffaa';
        this.ctx.fillStyle='#00ffaa';this.ctx.strokeStyle='#fff';this.ctx.lineWidth=2;
        for(const key in this.levelData){
            const[gx,gy]=key.split(',').map(Number);
            this.ctx.fillRect(gx*this.tileSize,gy*this.tileSize,this.tileSize,this.tileSize);
            this.ctx.strokeRect(gx*this.tileSize,gy*this.tileSize,this.tileSize,this.tileSize);
        }
        
        this.ctx.shadowBlur=0;
        for(let pt of this.particles){
            this.ctx.globalAlpha=Math.max(0,pt.life);
            this.ctx.fillStyle=pt.color;
            this.ctx.fillRect(pt.x,pt.y,6,6);
        }
        this.ctx.globalAlpha=1.0;
        
        if(this.state==='PLAYING'||this.state==='EDITING'){
            this.ctx.shadowBlur=15;this.ctx.shadowColor='#ff3366';
            this.ctx.fillStyle='#ff3366';
            this.ctx.fillRect(this.player.x,this.player.y,this.player.size,this.player.size);
            this.ctx.strokeStyle='#fff';this.ctx.strokeRect(this.player.x,this.player.y,this.player.size,this.player.size);
            this.ctx.shadowBlur=0;
        }
        this.ctx.restore();
    },
    loop(){this.update();this.draw();requestAnimationFrame(()=>this.loop());},
    exportLevel(){prompt("Copy Level Code:",btoa(JSON.stringify(this.levelData)));},
    importLevel(){
        const data=prompt("Paste Level Code:");
        if(data)try{this.levelData=JSON.parse(atob(data));}catch(e){alert("Invalid");}
    },
    importFromMenu(){
        const input=document.getElementById('menu-import-code'),data=input.value.trim();
        if(data)try{this.levelData=JSON.parse(atob(data));input.value='';this.changeState('PLAYING');}catch(e){alert("Invalid");}
    }
};
window.onload=()=>window.gameEngine.init();
</script>
</body>
</html>
