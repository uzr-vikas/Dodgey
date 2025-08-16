<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Dodgey</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: black;
      color: #fff;
      font-family: monospace;
      height: 100vh;
      overflow: hidden;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: flex-start;
    }
    #powerIndicators {
      height: 30px;
      margin-top: 8px;
      display: flex;
      gap: 10px;
    }
    canvas {
      background: url('https://i.postimg.cc/L8qXm6gX/file-0000000025a46230a3c72a8ce07ba7f7.png') center center / cover no-repeat;
      display: block;
      border: 2px inset #D3D3D3;
      max-width: 100%;
      height: auto;
    }
    #menu, #gameOverScreen {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      background:  url('https://i.postimg.cc/L8qXm6gX/file-0000000025a46230a3c72a8ce07ba7f7.png') center center / cover no-repeat;
      border: 2px outset #D3D3D3;
      z-index: 10;
      text-align: center;
      padding: 20px;
    }
    button {
      background: #0f0f0f;
      color: #e6e6e6;
      border: 2px ridge #D3D3D3;
      padding: 10px 20px;
      font-size: 18px;
      cursor: pointer;
      margin-top: 10px;
    }
    button:hover {
      background: #D3D3D3;
      color: #000;
    }
  </style>
</head>
<body>
  <div id="powerIndicators">
    <canvas id="powerCanvas" width="100" height="30"></canvas>
  </div>
  <canvas id="gameCanvas" width="350" height="600"></canvas>
  <div id="menu">
    <button onclick="startGame()">Play</button>
  </div>
  <div id="gameOverScreen" style="display:none;">
    <h2 id="scoreDisplay"></h2>
    <h3 id="avgScoreDisplay"></h3>
    <h3 id="highScoreDisplay"></h3>
    <h3 id="rankDisplay"></h3>
    <button onclick="startGame()">Play Again</button>
  </div>

  <audio id="bgMusic" src="https://files.catbox.moe/i1dv6x.mp3" loop></audio>
  <audio id="explodeSound" src="https://files.catbox.moe/56xco1.wav"></audio>

  <script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    const powerCanvas = document.getElementById("powerCanvas");
    const ptx = powerCanvas.getContext("2d");
    const bgMusic = document.getElementById("bgMusic");
    const explodeSound = document.getElementById("explodeSound");

    // preload sound
    explodeSound.volume = 0;
    explodeSound.play().then(() => {
      explodeSound.pause();
      explodeSound.currentTime = 0;
      explodeSound.volume = 1;
    });

    const playerImage = new Image();
    playerImage.src = "https://i.postimg.cc/4xyd92QN/1752221959984.png";
    const obstacleImage = new Image();
    obstacleImage.src = "https://i.postimg.cc/bJ33sQwh/IMG-20250705-130252.png";
    const laserPowerImage = new Image();
    laserPowerImage.src = "https://i.postimg.cc/KjjQLR1g/IMG-20250705-130146.png";
    const shieldPowerImage = new Image();
    shieldPowerImage.src = "https://i.postimg.cc/B6qmShsp/IMG-20250705-130219.png";
    const doublePowerImage = new Image();
    doublePowerImage.src = "https://i.postimg.cc/fR92BtBc/IMG-20250705-130039.png";
    const slowmoPowerImage = new Image();
    slowmoPowerImage.src = "https://i.postimg.cc/bvf6MyBK/IMG-20250705-130107.png";
    const boosterImage = new Image();
    boosterImage.src = "https://i.postimg.cc/5yjrqPXg/1751705238146.png";
    const crashImage = new Image();
    crashImage.src = "https://i.postimg.cc/tTGthcG3/IMG-20250705-194619.png";

    let player, obstacles = [], powerUps = [], lasers = [], stars = [];
    let score = 0, left = false, right = false, laserCooldown = 0;
    let shield = false, slowmo = false, doublePoints = false, laserActive = false;
    let baseSpeed = 2, currentSpeed = baseSpeed;
    let gameRunning = false, lastTime = 0;

    const playerSize = 42, obstacleSize = 30, powerUpSize = 20;

    function resetGame() {
      player = { x: canvas.width/2, y: canvas.height-60, width: playerSize, height: playerSize, speed: 5 };
      obstacles = []; powerUps = []; lasers = []; stars = [];
      score = 0; currentSpeed = baseSpeed;
      shield = slowmo = doublePoints = laserActive = false;
      laserCooldown = 0;
      drawPowerIndicators();
    }

    function drawPowerIndicators() {
      ptx.clearRect(0,0,powerCanvas.width,powerCanvas.height);
      const tri=10, gap=25, startX=10;
      const colors = [
        laserActive ? "#f00" : "#808080",
        shield ? "#0ff" : "#808080",
        slowmo ? "#f90" : "#808080",
        doublePoints ? "#fd1" : "#808080"
      ];
      colors.forEach((c,i)=>{
        const x = startX + i*gap, y=20;
        ptx.fillStyle=c;
        ptx.beginPath();
        ptx.moveTo(x,y-tri);
        ptx.lineTo(x-tri/2,y);
        ptx.lineTo(x+tri/2,y);
        ptx.closePath();
        ptx.fill();
      });
    }

    function drawPlayer() {
      if (shield) {
        ctx.shadowColor="#0ff";
        ctx.shadowBlur=25;
      }
      ctx.drawImage(playerImage, player.x-player.width/2,player.y,player.width,player.height);
      ctx.shadowBlur=0;

      // Custom booster settings:
      const boosterWidth=8, boosterHeight=12, boosterOffsetY=42, boosterSpacing=6.7;
      ctx.drawImage(boosterImage, player.x - boosterSpacing - boosterWidth/2, player.y + boosterOffsetY, boosterWidth, boosterHeight);
      ctx.drawImage(boosterImage, player.x + boosterSpacing - boosterWidth/2, player.y + boosterOffsetY, boosterWidth, boosterHeight);
      
     // Watermark (don't remove)
     ctx.fillStyle = "silver";
     ctx.font = "10px Monospace";
     ctx.textAlign = "right";
     ctx.fillText("FØZY", canvas.width - 10, canvas.height - 10);
    }

    function drawStars() {
      ctx.fillStyle = "#c0c3fa";
      stars.forEach(s => ctx.fillRect(s.x, s.y, 2,2));
    }

    function drawObstacles() {
      const now = performance.now();
      obstacles = obstacles.filter(o => !o.destroyed || (now - o.destroyTime < 1500));
      obstacles.forEach(o => {
        if (o.destroyed) ctx.drawImage(crashImage, o.x, o.y, obstacleSize, obstacleSize);
        else ctx.drawImage(obstacleImage, o.x, o.y, obstacleSize, obstacleSize);
      });
    }

    function drawPowerUps() {
      powerUps.forEach(p=>{
        let img = p.type==="laser"?laserPowerImage:
                  p.type==="shield"?shieldPowerImage:
                  p.type==="double"?doublePowerImage:
                  slowmoPowerImage;
        ctx.drawImage(img,p.x,p.y,powerUpSize,powerUpSize);
      });
    }

    function drawLasers(delta) {
      for (let i = lasers.length - 1; i >= 0; i--) {
       let l = lasers[i];
        l.y -= 10 * delta;

        // Add glow effect
        ctx.shadowColor = "#f00";
        ctx.shadowBlur = 5;

        ctx.fillStyle = "#ffcccc";
        ctx.fillRect(l.x, l.y, 4, 20);

        // Reset shadow after drawing
        ctx.shadowBlur = 0;

        let laserRemoved = false;
        for (let j = 0; j < obstacles.length; j++) {
          const o = obstacles[j];
          if (!o.destroyed) {
            const cx = o.x + obstacleSize / 2;
            const cy = o.y + obstacleSize / 2;
            const r = obstacleSize / 2;
            if (l.x > cx - r && l.x < cx + r && l.y < cy + r && l.y > cy - r) {
              o.destroyed = true;
              o.destroyTime = performance.now();
              explodeSound.currentTime = 0; explodeSound.play();
              laserRemoved = true;
              break;
            }
          }
        }

        if (laserRemoved || l.y < -20) {
          lasers.splice(i, 1);
        }
      }
    }

    function updateEntities(delta) {
      const speed = currentSpeed * delta;
      obstacles.forEach(o=>o.y+=speed);
      powerUps.forEach(p=>p.y+=speed);
      stars.forEach(s=>s.y+=s.speed*delta);
      stars = stars.filter(s=>s.y<canvas.height);
    }

    function spawnEntities() {
      if(Math.random()<0.05) obstacles.push({ x:Math.random()*(canvas.width-obstacleSize), y:0, destroyed:false });
      if(Math.random()<0.02) {
        const types=["shield","slowmo","double","laser"];
        powerUps.push({ x:Math.random()*(canvas.width-powerUpSize), y:0, type: types[Math.floor(Math.random()*4)] });
      }
      if(stars.length<100) stars.push({ x:Math.random()*canvas.width, y:0, speed:Math.random()*2+1 });
    }

    function checkCollision(p,c){
      const px=p.x-p.width/2, py=p.y, pw=p.width, ph=p.height;
      const cx=c.x+obstacleSize/2, cy=c.y+obstacleSize/2, r=obstacleSize/2;
      const cx0 = Math.max(px,Math.min(cx,px+pw));
      const cy0 = Math.max(py,Math.min(cy,py+ph));
      const dx=cx-cx0, dy=cy-cy0;
      return dx*dx+dy*dy < r*r;
    }

    function activatePowerUp(type){
      if(type==="shield"){ shield=true; setTimeout(()=>{shield=false;drawPowerIndicators()},8000); }
      else if(type==="slowmo"){ slowmo=true; currentSpeed=1; setTimeout(()=>{slowmo=false;currentSpeed=baseSpeed;drawPowerIndicators()},4000); }
      else if(type==="double"){ doublePoints=true; setTimeout(()=>{doublePoints=false;drawPowerIndicators()},8000); }
      else if(type==="laser"){ laserActive=true; setTimeout(()=>{laserActive=false;drawPowerIndicators()},8000); }
      drawPowerIndicators();
    }

    function drawScore() {
     ctx.fillStyle = doublePoints ? "#fd1" : "#fff";
     ctx.font = "16px monospace";
     ctx.textAlign = "left";   // <--- Added
     ctx.fillText(`Score: ${score}`, 10, 20);
    }
    
    function gameLoop(timestamp){
      if(!gameRunning)return;
      const delta=(timestamp-lastTime)/16.666;
      lastTime=timestamp;

      ctx.clearRect(0,0,canvas.width,canvas.height);
      spawnEntities();
      drawStars();
      drawPlayer();
      drawObstacles();
      drawPowerUps();
      drawLasers(delta);
      updateEntities(delta);
      drawScore();

      if(left) player.x-=player.speed;
      if(right) player.x+=player.speed;
      player.x = Math.max(player.width/2,Math.min(canvas.width-player.width/2,player.x));

      if(laserActive && laserCooldown<=0){ lasers.push({ x:player.x-2, y:player.y }); laserCooldown=8; }
      else if(laserCooldown>0){ laserCooldown--; }

      for(let o of obstacles){
        if(!o.destroyed && checkCollision(player,o)){
          o.destroyed=true; o.destroyTime=performance.now();
          explodeSound.currentTime=0; explodeSound.play();
          if(shield){
            shield=false;
            obstacles.splice(obstacles.indexOf(o),1);
            drawPowerIndicators();
          } else {
            gameRunning=false;
            ctx.drawImage(crashImage, o.x,o.y, obstacleSize+35, obstacleSize+35);
            setTimeout(endGame,500);
            return;
          }
        }
      }

      for(let p of powerUps){
        if(checkCollision(player,p)){
          activatePowerUp(p.type);
          powerUps.splice(powerUps.indexOf(p),1);
        }
      }

      score += doublePoints ? 2 : 1;
      requestAnimationFrame(gameLoop);
    }

    function startGame(){
      resetGame();
      bgMusic.play().catch(()=>{});
      gameRunning=true;
      document.getElementById("menu").style.display="none";
      document.getElementById("gameOverScreen").style.display="none";
      lastTime=performance.now();
      requestAnimationFrame(gameLoop);
    }

    function endGame(){
      bgMusic.pause();
      const scores = JSON.parse(localStorage.getItem("scores")||"[]");
      scores.push(score);
      if(scores.length>10)scores.shift();
      localStorage.setItem("scores",JSON.stringify(scores));
      const avg = Math.floor(scores.reduce((a,b)=>a+b,0)/scores.length);
      let high = parseInt(localStorage.getItem("highScore")||"0");
      if(score>high){ high=score; localStorage.setItem("highScore",high); }
      document.getElementById("scoreDisplay").innerText = `Score: ${score}`;
      document.getElementById("avgScoreDisplay").innerText = `Average: ${avg}`;
      document.getElementById("highScoreDisplay").innerText = `High Score: ${high}`;
      document.getElementById("rankDisplay").innerText = (avg>=9001?"CEO":avg>=4751?"Cosmic General":avg>=3751?"Cosmic Captain":avg>=3251?"Cosmic Leftinent":avg>=2501?"Cosmic Sargent":avg>=2001?"Cosmic Cadet":"Cosmic Rider");
      document.getElementById("gameOverScreen").style.display="block";
    }

    window.addEventListener("keydown",e=>{ if(e.key==="ArrowLeft")left=true; if(e.key==="ArrowRight")right=true; });
    window.addEventListener("keyup",e=>{ if(e.key==="ArrowLeft")left=false; if(e.key==="ArrowRight")right=false; });
    window.addEventListener("touchstart",e=>{ left = e.touches[0].clientX < window.innerWidth/2; right = !left; });
    window.addEventListener("touchend", ()=>{ left = right = false; });

    document.getElementById("menu").style.display="block";
    window.addEventListener("blur",()=>gameRunning=false);
    window.addEventListener("focus",()=>{ if(!gameRunning && score>0) startGame(); });
  </script>
</body>
</html>
