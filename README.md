<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Safe Mines — Interactive Game</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <div class="container">
    <header>
      <h1>Safe Mines</h1>
      <div class="controls">
        <label>Size:
          <select id="sizeSelect">
            <option value="8">8 × 8</option>
            <option value="10" selected>10 × 10</option>
            <option value="12">12 × 12</option>
          </select>
        </label>
        <label>Mines:
          <select id="minesSelect">
            <option value="10">10</option>
            <option value="15" selected>15</option>
            <option value="20">20</option>
          </select>
        </label>
        <button id="newBtn">New Game</button>
      </div>
    </header>

    <main>
      <div class="info">
        <div>Time: <span id="timer">0</span>s</div>
        <div>Mines left: <span id="minesLeft">0</span></div>
      </div>

      <div id="board" class="board" aria-label="Game board"></div>

      <div class="footer">
        <p>Left-click to reveal. Right-click (or long-press) to flag.</p>
        <p class="note">This is a learning project — feel free to modify it.</p>
      </div>
    </main>
  </div>

  <script src="script.js"></script>
</body>
</html>
:root{
  --bg: #0f172a;
  --card: #0b1220;
  --accent: #06b6d4;
  --muted: #94a3b8;
  --safe: #10b981;
}
*{box-sizing:border-box}
html,body{height:100%}
body{
  margin:0;
  font-family:Inter,system-ui,Segoe UI,Roboto,Arial;
  background: linear-gradient(180deg,#071027 0%, #0b1320 100%);
  color:#e6eef8;
  display:flex;
  align-items:center;
  justify-content:center;
  padding:20px;
}
.container{
  width:100%;
  max-width:960px;
  background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
  border-radius:14px;
  padding:18px;
  box-shadow:0 8px 30px rgba(2,6,23,0.7);
}
header{
  display:flex;
  align-items:center;
  justify-content:space-between;
  gap:12px;
}
header h1{margin:0;font-size:1.4rem;color:var(--accent)}
.controls{display:flex;gap:8px;align-items:center}
.controls label{font-size:0.9rem;color:var(--muted)}
.controls select, .controls button{
  padding:6px 8px;border-radius:8px;border:1px solid rgba(255,255,255,0.05);
  background:rgba(255,255,255,0.02);color:inherit;
}
.controls button{cursor:pointer}
.info{display:flex;gap:18px;justify-content:center;margin:12px 0;color:var(--muted)}
.board{
  display:grid;
  gap:6px;
  justify-content:center;
  padding:12px;
  background:rgba(255,255,255,0.02);
  border-radius:10px;
}
.cell{
  width:36px;height:36px;
  display:flex;align-items:center;justify-content:center;
  background:linear-gradient(180deg,#0b2340,#081427);
  border-radius:6px;
  cursor:pointer;
  user-select:none;
  font-weight:600;
  color:#dbeafe;
  font-size:14px;
  border:1px solid rgba(255,255,255,0.03);
}
.cell.revealed{
  background:linear-gradient(180deg,#cfe8ff,#e6f2ff);
  color:#0b1220;
  cursor:default;
  border:1px solid rgba(11,18,32,0.06);
}
.cell.mine {
  background: linear-gradient(180deg,#ff4444,#cc0000);
  color: white;
}
.cell.flag{background:linear-gradient(180deg,#ffd966,#ffb84d)}
.footer{margin-top:12px;text-align:center;color:var(--muted);font-size:0.9rem}
.note{font-size:0.8rem;color:#8fb0c6}
@media (max-width:420px){
  .cell{width:28px;height:28px;font-size:12px}
}
// Safe Mines - simple Mines-like game
// Click to reveal, right-click (or long press) to flag.

const boardEl = document.getElementById('board');
const timerEl = document.getElementById('timer');
const minesLeftEl = document.getElementById('minesLeft');
const newBtn = document.getElementById('newBtn');
const sizeSelect = document.getElementById('sizeSelect');
const minesSelect = document.getElementById('minesSelect');

let size = parseInt(sizeSelect.value);
let minesCount = parseInt(minesSelect.value);
let cells = [];
let minePositions = new Set();
let revealedCount = 0;
let flaggedCount = 0;
let timer = null;
let seconds = 0;
let gameOver = false;
let longPressTimer = null;

function newGame() {
  size = parseInt(sizeSelect.value);
  minesCount = parseInt(minesSelect.value);
  boardEl.innerHTML = '';
  cells = [];
  minePositions.clear();
  revealedCount = 0;
  flaggedCount = 0;
  seconds = 0;
  timerEl.textContent = seconds;
  minesLeftEl.textContent = minesCount;
  gameOver = false;
  clearInterval(timer);
  buildGrid(size);
  placeMines();
  calculateNumbers();
}

function buildGrid(n) {
  boardEl.style.gridTemplateColumns = `repeat(${n}, auto)`;
  for (let r=0; r<n; r++) {
    for (let c=0; c<n; c++) {
      const idx = r*n + c;
      const cell = document.createElement('button');
      cell.className = 'cell';
      cell.dataset.idx = idx;
      cell.dataset.r = r;
      cell.dataset.c = c;
      cell.dataset.val = '0';
      cell.setAttribute('aria-label', `Cell ${r+1}-${c+1}`);
      addCellListeners(cell);
      boardEl.appendChild(cell);
      cells.push(cell);
    }
  }
}

function addCellListeners(cell){
  cell.addEventListener('click', onClick);
  cell.addEventListener('contextmenu', onRightClick);
  // simple long-press support for mobile (800ms)
  cell.addEventListener('touchstart', e=>{
    if (gameOver) return;
    longPressTimer = setTimeout(()=> {
      flagCell(e);
    },800);
  }, {passive:true});
  cell.addEventListener('touchend', e=>{
    clearTimeout(longPressTimer);
  });
}

function onClick(e){
  if (gameOver) return;
  const idx = +thisOrTarget(e).dataset.idx;
  if (!timer) startTimer();
  revealCell(idx);
}

function onRightClick(e){
  e.preventDefault();
  if (gameOver) return;
  if (!timer) startTimer();
  flagCell(e);
}

function thisOrTarget(e){
  return e.currentTarget || e.target;
}

function flagCell(e){
  const el = thisOrTarget(e);
  const idx = +el.dataset.idx;
  if (cells[idx].classList.contains('revealed')) return;
  if (cells[idx].classList.contains('flag')) {
    cells[idx].classList.remove('flag');
    flaggedCount--;
  } else {
    cells[idx].classList.add('flag');
    flaggedCount++;
  }
  minesLeftEl.textContent = Math.max(0, minesCount - flaggedCount);
  checkWin();
}

function placeMines(){
  const total = size*size;
  while (minePositions.size < minesCount) {
    const pos = Math.floor(Math.random()*total);
    minePositions.add(pos);
    cells[pos].dataset.val = 'M';
  }
}

function calculateNumbers(){
  const n=size;
  for (let r=0;r<n;r++){
    for (let c=0;c<n;c++){
      const idx = r*n + c;
      if (cells[idx].dataset.val === 'M') {
        cells[idx].classList.add('mine'); // hidden style only for later reveal
        continue;
      }
      let count = 0;
      for (let dr=-1; dr<=1; dr++){
        for (let dc=-1; dc<=1; dc++){
          if (dr===0 && dc===0) continue;
          const rr = r+dr, cc = c+dc;
          if (rr<0||cc<0||rr>=n||cc>=n) continue;
          const neighborIdx = rr*n+cc;
          if (cells[neighborIdx].dataset.val === 'M') count++;
        }
      }
      cells[idx].dataset.val = String(count);
    }
  }
}

function revealCell(idx){
  const el = cells[idx];
  if (!el || el.classList.contains('revealed') || el.classList.contains('flag')) return;
  el.classList.add('revealed');
  const val = el.dataset.val;
  if (val === 'M') {
    // reveal mines and end
    el.classList.add('mine');
    loseGame();
    return;
  }
  revealedCount++;
  if (val !== '0') {
    el.textContent = val;
    el.style.color = colorForNumber(+val);
  } else {
    // flood fill neighbors
    el.textContent = '';
    const n=size;
    const r = +el.dataset.r, c = +el.dataset.c;
    for (let dr=-1; dr<=1; dr++){
      for (let dc=-1; dc<=1; dc++){
        if (dr===0 && dc===0) continue;
        const rr=r+dr, cc=c+dc;
        if (rr<0||cc<0||rr>=n||cc>=n) continue;
        revealCell(rr*n+cc);
      }
    }
  }
  checkWin();
}

function colorForNumber(num){
  switch(num){
    case 1: return '#1e40af';
    case 2: return '#0f766e';
    case 3: return '#b91c1c';
    case 4: return '#7c2d12';
    case 5: return '#6b21a8';
    default: return '#0b1220';
  }
}

function loseGame(){
  gameOver = true;
  clearInterval(timer);
  // reveal all mines
  minePositions.forEach(i=>{
    const c = cells[i];
    if (!c.classList.contains('revealed')) {
      c.classList.add('revealed','mine');
      c.textContent = '💣';
    }
  });
  setTimeout(()=> alert('Boom! You hit a mine — Game over.'), 100);
}

function checkWin(){
  const total = size*size;
  if (revealedCount + minePositions.size === total) {
    gameOver = true;
    clearInterval(timer);
    setTimeout(()=> alert(`Congratulations! You cleared the board in ${seconds}s.`), 80);
    // optionally reveal flags
  }
}

function startTimer(){
  timer = setInterval(()=>{
    seconds++;
    timerEl.textContent = seconds;
  },1000);
}

newBtn.addEventListener('click', newGame);
sizeSelect.addEventListener('change', ()=> {
  // adjust default mines when size changes
  const s = parseInt(sizeSelect.value);
  const area = s*s;
  // scale mines proportionally
  minesSelect.value = Math.max(5, Math.round(area*0.15));
});
minesSelect.addEventListener('change', ()=>{ /* nothing */});

newGame();
