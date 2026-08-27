<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Circuit Snake</title>
<link rel="manifest" href="manifest.json">
<meta name="theme-color" content="#0b1220">
<link rel="icon" href="icon-192.png">
<link rel="apple-touch-icon" href="icon-192.png">
<style>
  :root{
    --bg: #0b1220;
    --grid: #101a2e;
    --accent: #5eead4;
    --accent2: #f472b6;
    --text: #e6edf7;
    --muted: #7c8aa5;
  }
  *{box-sizing:border-box;}
  body{
    margin:0;
    min-height:100vh;
    display:flex;
    flex-direction:column;
    align-items:center;
    justify-content:center;
    background:
      radial-gradient(circle at 20% 10%, #14213a 0%, transparent 40%),
      radial-gradient(circle at 80% 90%, #1b1030 0%, transparent 40%),
      var(--bg);
    font-family: 'Courier New', ui-monospace, monospace;
    color: var(--text);
    padding: 24px;
  }
  h1{
    font-size: clamp(1.5rem, 5vw, 2.2rem);
    letter-spacing: 0.15em;
    margin: 0 0 4px;
    text-transform: uppercase;
  }
  h1 span{ color: var(--accent); }
  .sub{
    color: var(--muted);
    font-size: 0.8rem;
    letter-spacing: 0.1em;
    margin-bottom: 18px;
    text-align:center;
  }
  .hud{
    display:flex;
    gap: 24px;
    margin-bottom: 12px;
    font-size: 0.95rem;
    letter-spacing: 0.08em;
  }
  .hud div span{ color: var(--accent); font-weight:bold; }
  #wrap{
    position: relative;
    border: 2px solid #253150;
    box-shadow: 0 0 0 6px #0b1220, 0 0 40px rgba(94,234,212,0.08);
    border-radius: 6px;
    overflow:hidden;
  }
  canvas{
    display:block;
    background: var(--grid);
    image-rendering: pixelated;
  }
  #overlay{
    position:absolute; inset:0;
    display:flex; flex-direction:column; align-items:center; justify-content:center;
    background: rgba(11,18,32,0.85);
    text-align:center;
    padding: 20px;
  }
  #overlay h2{ margin:0 0 8px; letter-spacing:0.1em; color: var(--accent2); }
  #overlay p{ color: var(--muted); margin:4px 0 16px; font-size:0.85rem; }
  button{
    background: var(--accent);
    color: #06251f;
    border:none;
    padding: 10px 22px;
    font-family: inherit;
    font-weight:bold;
    letter-spacing: 0.08em;
    border-radius: 4px;
    cursor:pointer;
    font-size: 0.85rem;
    text-transform: uppercase;
  }
  button:hover{ background:#8bf3e2; }
  .hint{
    margin-top: 14px;
    color: var(--muted);
    font-size: 0.75rem;
    letter-spacing: 0.06em;
    text-align:center;
  }
  .dpad{
    display:none;
    margin-top:16px;
    grid-template-columns: repeat(3, 48px);
    grid-template-rows: repeat(2, 48px);
    gap:6px;
  }
  .dpad button{ padding:0; font-size:1.1rem; }
  @media (max-width: 560px){
    .dpad{ display:grid; }
  }
</style>
</head>
<body>
  <h1>Circuit <span>Snake</span></h1>
  <div class="sub">Grow the current. Don't cross the wire.</div>
  <div class="hud">
    <div>Score <span id="score">0</span></div>
    <div>Best <span id="best">0</span></div>
  </div>
  <div id="wrap">
    <canvas id="game" width="400" height="400"></canvas>
    <div id="overlay">
      <h2 id="overlayTitle">Ready?</h2>
      <p id="overlayText">Arrow keys or swipe to move.</p>
      <button id="startBtn">Start Game</button>
    </div>
  </div>
  <div class="hint">Arrow keys / WASD to steer · Space to pause</div>
  <div class="dpad">
    <div></div><button id="up">▲</button><div></div>
    <button id="left">◀</button><button id="down">▼</button><button id="right">▶</button>
  </div>

<script>
(function(){
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const cell = 20;
  const cols = canvas.width / cell;
  const rows = canvas.height / cell;

  let snake, dir, nextDir, food, score, best, alive, paused, loopId, speed;

  const scoreEl = document.getElementById('score');
  const bestEl = document.getElementById('best');
  const overlay = document.getElementById('overlay');
  const overlayTitle = document.getElementById('overlayTitle');
  const overlayText = document.getElementById('overlayText');
  const startBtn = document.getElementById('startBtn');

  best = parseInt(localStorage_safe_get() || '0', 10);
  bestEl.textContent = best;

  function localStorage_safe_get(){
    // Artifact-safe: no browser storage. Use in-memory only.
    return null;
  }

  function reset(){
    snake = [
      {x: 8, y: 10},
      {x: 7, y: 10},
      {x: 6, y: 10}
    ];
    dir = {x:1, y:0};
    nextDir = {x:1, y:0};
    score = 0;
    speed = 120;
    alive = true;
    paused = false;
    placeFood();
    scoreEl.textContent = score;
  }

  function placeFood(){
    let valid = false;
    while(!valid){
      food = {
        x: Math.floor(Math.random()*cols),
        y: Math.floor(Math.random()*rows)
      };
      valid = !snake.some(s => s.x===food.x && s.y===food.y);
    }
  }

  function draw(){
    ctx.clearRect(0,0,canvas.width,canvas.height);

    // grid dots
    ctx.fillStyle = '#16223f';
    for(let x=0;x<cols;x++){
      for(let y=0;y<rows;y++){
        ctx.fillRect(x*cell+cell/2-1, y*cell+cell/2-1, 2, 2);
      }
    }

    // food
    ctx.fillStyle = '#f472b6';
    ctx.shadowColor = '#f472b6';
    ctx.shadowBlur = 10;
    ctx.beginPath();
    ctx.arc(food.x*cell+cell/2, food.y*cell+cell/2, cell/2-3, 0, Math.PI*2);
    ctx.fill();
    ctx.shadowBlur = 0;

    // snake
    snake.forEach((s, i) => {
      const t = i / snake.length;
      ctx.fillStyle = i===0 ? '#5eead4' : `rgba(94,234,212,${0.9 - t*0.5})`;
      ctx.fillRect(s.x*cell+1, s.y*cell+1, cell-2, cell-2);
    });
  }

  function tick(){
    if(!alive || paused) return;
    dir = nextDir;
    const head = {x: snake[0].x + dir.x, y: snake[0].y + dir.y};

    if(head.x < 0 || head.x >= cols || head.y < 0 || head.y >= rows || snake.some(s => s.x===head.x && s.y===head.y)){
      gameOver();
      return;
    }

    snake.unshift(head);

    if(head.x === food.x && head.y === food.y){
      score += 10;
      scoreEl.textContent = score;
      if(score > best){ best = score; bestEl.textContent = best; }
      placeFood();
      if(speed > 60) speed -= 3;
    } else {
      snake.pop();
    }

    draw();
  }

  function gameOver(){
    alive = false;
    overlayTitle.textContent = 'Circuit Broken';
    overlayText.textContent = `Score: ${score} · Best: ${best}`;
    startBtn.textContent = 'Try Again';
    overlay.style.display = 'flex';
  }

  let timer;
  function loop(){
    tick();
    timer = setTimeout(loop, speed);
  }

  function startGame(){
    reset();
    overlay.style.display = 'none';
    draw();
    clearTimeout(timer);
    loop();
  }

  startBtn.addEventListener('click', startGame);

  document.addEventListener('keydown', (e) => {
    const k = e.key.toLowerCase();
    if(k === ' '){ 
      if(alive){ paused = !paused; if(!paused) loop(); }
      e.preventDefault();
      return;
    }
    if((k==='arrowup'||k==='w') && dir.y===0){ nextDir={x:0,y:-1}; }
    else if((k==='arrowdown'||k==='s') && dir.y===0){ nextDir={x:0,y:1}; }
    else if((k==='arrowleft'||k==='a') && dir.x===0){ nextDir={x:-1,y:0}; }
    else if((k==='arrowright'||k==='d') && dir.x===0){ nextDir={x:1,y:0}; }
  });

  // touch dpad
  function bindDpad(id, dx, dy){
    document.getElementById(id).addEventListener('click', () => {
      if(dx!==0 && dir.x===0) nextDir={x:dx,y:0};
      if(dy!==0 && dir.y===0) nextDir={x:0,y:dy};
    });
  }
  bindDpad('up',0,-1);
  bindDpad('down',0,1);
  bindDpad('left',-1,0);
  bindDpad('right',1,0);

  // swipe support
  let touchStart = null;
  canvas.addEventListener('touchstart', e => {
    touchStart = e.touches[0];
  });
  canvas.addEventListener('touchend', e => {
    if(!touchStart) return;
    const dx = e.changedTouches[0].clientX - touchStart.clientX;
    const dy = e.changedTouches[0].clientY - touchStart.clientY;
    if(Math.abs(dx) > Math.abs(dy)){
      if(dx > 20 && dir.x===0) nextDir={x:1,y:0};
      else if(dx < -20 && dir.x===0) nextDir={x:-1,y:0};
    } else {
      if(dy > 20 && dir.y===0) nextDir={x:0,y:1};
      else if(dy < -20 && dir.y===0) nextDir={x:0,y:-1};
    }
    touchStart = null;
  });

  reset();
  draw();

  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker.register('sw.js').catch(() => {});
    });
  }
})();
</script>
</body>
</html>
