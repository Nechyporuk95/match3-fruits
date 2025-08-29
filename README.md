<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Фрукты 3 в ряд — 50 уровней</title>
<style>
body {
  margin:0;
  font-family: Arial, sans-serif;
  background: url('https://images.unsplash.com/photo-1507525428034-b723cf961d3e?auto=format&fit=crop&w=1350&q=80') no-repeat center center fixed;
  background-size: cover;
  display:flex;
  flex-direction:column;
  align-items:center;
  justify-content:center;
  height:100vh;
}
#menu, #game, #levelSelect, #scores {
  display:none;
  text-align:center;
  background: rgba(255,255,255,0.2);
  padding: 20px;
  border-radius: 15px;
}
.btn {
  background: #ff7f50;
  border: none;
  padding: 10px 20px;
  margin: 5px;
  border-radius: 12px;
  font-size: 18px;
  color: white;
  cursor: pointer;
  transition: 0.3s;
}
.btn:hover { background: #ff4500; }
#board {
  display: grid;
  gap: 2px;
  margin: 20px auto;
  justify-content: center;
}
.cell {
  display:flex;
  align-items:center;
  justify-content:center;
  font-size:28px;
  cursor:pointer;
  user-select:none;
  position: relative;
  transition: transform 0.4s ease, opacity 0.4s ease, top 0.4s ease;
}
.cell.match {
  animation: vanish 0.6s forwards;
}
@keyframes vanish {
  0% { transform: scale(1) rotate(0deg); opacity:1; }
  50% { transform: scale(1.5) rotate(180deg); opacity:0.7; }
  100% { transform: scale(0) rotate(360deg); opacity:0; }
}
#score, #task {
  font-size:20px;
  margin:10px;
}
/* Подсказка */
@keyframes hintBlink {
  0% { transform: scale(1); }
  50% { transform: scale(1.3); }
  100% { transform: scale(1); }
}
.cell.hint { animation: hintBlink 0.8s infinite; }

/* Таблица рекордов */
#scores table {
  width:100%;
  border-collapse: collapse;
}
#scores th, #scores td {
  padding:5px;
  border-bottom:1px solid #ccc;
  color:#fff;
}
</style>
</head>
<body>
<div id="menu">
  <h1>Фрукты 3 в ряд</h1>
  <button class="btn" onclick="startGame()">Начать игру</button>
  <button class="btn" onclick="showLevelSelect()">Выбрать уровень</button>
  <button class="btn" onclick="showScores()">Таблица рекордов</button>
</div>
<div id="levelSelect">
  <h2>Выбор уровня</h2>
  <div id="levels"></div>
  <button class="btn" onclick="backToMenu()">Назад</button>
</div>
<div id="game">
  <div id="score">Очки: 0</div>
  <div id="task">Задание: </div>
  <div id="board"></div>
  <button class="btn" onclick="backToMenu()">В меню</button>
</div>
<div id="scores">
  <h2>Таблица рекордов</h2>
  <table>
    <thead><tr><th>Место</th><th>Очки</th></tr></thead>
    <tbody id="scoreTable"></tbody>
  </table>
  <button class="btn" onclick="backToMenu()">Назад</button>
</div>
<audio id="matchSound" src="https://actions.google.com/sounds/v1/cartoon/wood_plank_flicks.ogg"></audio>

<script>
const fruits = ["🍎","🍌","🍇","🍊","🍉","🍒","🍓","🍍"];
const specials = ["🍏","🥝"];
let board=[], size=6, score=0, selected=null, currentLevel=1, collected=0, idleTimer=null;
const boardEl=document.getElementById("board");
const scoreEl=document.getElementById("score");
const taskEl=document.getElementById("task");
const matchSound=document.getElementById("matchSound");

// Уровни
const LEVELS=[];
for(let i=1;i<=50;i++){
  let sz=6+Math.floor(i/10);
  let type='points';
  let fruit=null;
  let amount=100+i*10;
  if(i%5===2){ type='collect'; fruit=fruits[Math.floor(Math.random()*fruits.length)]; amount=20+Math.floor(i/2); }
  else if(i%5===3){ type='bonus'; fruit=i%2===0?'🍏':'🥝'; amount=5+Math.floor(i/3); }
  else if(i%5===4){ type='collect'; fruit=fruits[Math.floor(Math.random()*fruits.length)]; amount=25+Math.floor(i/2); }
  LEVELS.push({size:sz, task:{type, fruit, amount}});
}

// рандом фрукт (с приоритетом на задания)
function randomFruit(){
  const t=LEVELS[currentLevel-1].task;
  let r=Math.random();
  if(r<0.05) return "🍏";
  if(r<0.1) return "🥝";
  if(t.type==='collect' && Math.random()<0.25) return t.fruit;
  return fruits[Math.floor(Math.random()*fruits.length)];
}

function initBoard(){
  do {
    size=LEVELS[currentLevel-1].size;
    boardEl.style.gridTemplateColumns=`repeat(${size},50px)`;
    boardEl.style.gridTemplateRows=`repeat(${size},50px)`;
    board=[]; boardEl.innerHTML='';
    for(let r=0;r<size;r++){
      let row=[];
      for(let c=0;c<size;c++){
        let f=randomFruit();
        row.push(f);
        const cell=document.createElement("div");
        cell.classList.add("cell");
        cell.textContent=f;
        cell.onclick=()=>{selectCell(r,c,cell); resetIdleTimer();};
        boardEl.appendChild(cell);
      }
      row.length && board.push(row);
    }
  } while(findMoves().length===0);
  score=0; collected=0;
  scoreEl.textContent="Очки: 0"; showTask(); resetIdleTimer();
}

function showTask(){
  const t=LEVELS[currentLevel-1].task;
  if(t.type==='points') taskEl.textContent=`Задание: набрать ${t.amount} очков`;
  else taskEl.textContent=`Задание: ${t.type==='collect'?'собрать':'активировать'} ${t.amount} ${t.fruit} (собрано: ${collected}/${t.amount})`;
}

function selectCell(r,c,cell){
  if(!selected){selected={r,c,cell}; cell.style.transform='scale(1.2)';}
  else{
    let sr=selected.r, sc=selected.c;
    if(Math.abs(sr-r)+Math.abs(sc-c)===1){ swap(sr,sc,r,c); if(!checkMatches()) swap(sr,sc,r,c); }
    selected.cell.style.transform='scale(1)'; selected=null;
  }
}

function swap(r1,c1,r2,c2){ let t=board[r1][c1]; board[r1][c1]=board[r2][c2]; board[r2][c2]=t; renderBoard(); }

function checkMatches(){
  let matched=[];
  for(let r=0;r<size;r++) for(let c=0;c<size-2;c++){ let f=board[r][c]; if(f&&f===board[r][c+1]&&f===board[r][c+2]) matched.push([r,c],[r,c+1],[r,c+2]); }
  for(let c=0;c<size;c++) for(let r=0;r<size-2;r++){ let f=board[r][c]; if(f&&f===board[r+1][c]&&f===board[r+2][c]) matched.push([r,c],[r+1,c],[r+2,c]); }
  if(matched.length>0){
    matched.forEach(([r,c])=>{
      let f=board[r][c]; const t=LEVELS[currentLevel-1].task;
      if(t.type==='collect' && f===t.fruit) collected++;
      if(t.type==='bonus' && f===t.fruit) collected++;
      if(f==='🍏'){ for(let col=0;col<size;col++) matched.push([r,col]); score+=100; }
      if(f==='🥝'){ for(let row=0;row<size;row++) matched.push([row,c]); score+=100; }
    });
    matched.forEach(([r,c])=>{ const cell=boardEl.children[r*size+c]; cell.classList.add('match'); });
    setTimeout(()=>{
      matched.forEach(([r,c])=>board[r][c]=null);
      dropFruits();
      renderBoard();
      if(findMoves().length===0) reshuffleBoard();
      checkTask();
    },600);
    score+=matched.length*10; scoreEl.textContent="Очки: "+score; showTask(); matchSound.play();
    return true;
  }
  return false;
}

function dropFruits(){
  for(let c=0;c<size;c++){
    let empty=0;
    for(let r=size-1;r>=0;r--){
      if(!board[r][c]) empty++;
      else if(empty>0){ board[r+empty][c]=board[r][c]; board[r][c]=null; }
    }
    for(let r=0;r<empty;r++){ board[r][c]=randomFruit(); }
  }
}

function renderBoard(){
  for(let r=0;r<size;r++){
    for(let c=0;c<size;c++){
      const cell=boardEl.children[r*size+c];
      cell.textContent=board[r][c];
      cell.classList.remove('match','hint');
      cell.style.transform='scale(1)';
      cell.style.top="0px";
      if(!board[r][c]) continue;
      if(cell.style.opacity==="0"){
        cell.style.top="-60px";
        cell.style.opacity="1";
        setTimeout(()=>{cell.style.top="0px";},50);
      }
    }
  }
}

function reshuffleBoard(){
  let flat=board.flat().filter(Boolean);
  do {
    flat.sort(()=>Math.random()-0.5);
    for(let r=0;r<size;r++) for(let c=0;c<size;c++) board[r][c]=flat[r*size+c];
  } while(findMoves().length===0);
  renderBoard();
}

function checkTask(){
  const t=LEVELS[currentLevel-1].task; let done=false;
  if(t.type==='points' && score>=t.amount) done=true;
  if((t.type==='collect'||t.type==='bonus') && collected>=t.amount) done=true;
  if(done){ saveScore(score); alert('Уровень пройден!'); currentLevel++; if(currentLevel>LEVELS.length) currentLevel=1; initBoard(); }
}

// --- Подсказки ---
function findMoves(){ let m=[]; function swapTmp(r1,c1,r2,c2){ let t=board[r1][c1]; board[r1][c1]=board[r2][c2]; board[r2][c2]=t; }
  for(let r=0;r<size;r++)for(let c=0;c<size;c++){ if(c<size-1){swapTmp(r,c,r,c+1); if(hasMatch()) m.push([[r,c],[r,c+1]]); swapTmp(r,c,r,c+1);} if(r<size-1){swapTmp(r,c,r+1,c); if(hasMatch()) m.push([[r,c],[r+1,c]]); swapTmp(r,c,r+1,c);} }
  return m; }
function hasMatch(){ for(let r=0;r<size;r++) for(let c=0;c<size-2;c++){ let f=board[r][c]; if(f&&f===board[r][c+1]&&f===board[r][c+2]) return true; } for(let c=0;c<size;c++) for(let r=0;r<size-2;r++){ let f=board[r][c]; if(f&&f===board[r+1][c]&&f===board[r+2][c]) return true; } return false; }
function resetIdleTimer(){ clearTimeout(idleTimer); clearHint(); idleTimer=setTimeout(showHint,5000); }
function showHint(){ let moves=findMoves(); if(moves.length>0){ let move=moves[Math.floor(Math.random()*moves.length)]; move.forEach(([r,c])=>boardEl.children[r*size+c].classList.add("hint")); } }
function clearHint(){ [...boardEl.children].forEach(cell=>cell.classList.remove("hint")); }

// --- Рекорды ---
function saveScore(s){ let scores=JSON.parse(localStorage.getItem("scores")||"[]"); scores.push(s); scores.sort((a,b)=>b-a); scores=scores.slice(0,5); localStorage.setItem("scores",JSON.stringify(scores)); }
function showScores(){ document.getElementById("menu").style.display='none'; document.getElementById("scores").style.display='block'; let scores=JSON.parse(localStorage.getItem("scores")||"[]"); let tbody=document.getElementById("scoreTable"); tbody.innerHTML=''; scores.forEach((s,i)=>{ tbody.innerHTML+=`<tr><td>${i+1}</td><td>${s}</td></tr>`; }); }

// --- Меню ---
function startGame(){ document.getElementById('menu').style.display='none'; document.getElementById('levelSelect').style.display='none'; document.getElementById('scores').style.display='none'; document.getElementById('game').style.display='block'; initBoard(); }
function showLevelSelect(){ document.getElementById('menu').style.display='none'; document.getElementById('levelSelect').style.display='block'; const levelsDiv=document.getElementById('levels'); levelsDiv.innerHTML=''; for(let i=1;i<=50;i++){ const btn=document.createElement('button'); btn.className='btn'; btn.textContent='Уровень '+i; btn.onclick=()=>{currentLevel=i; initBoard(); document.getElementById('levelSelect').style.display='none'; document.getElementById('game').style.display='block';}; levelsDiv.appendChild(btn);} }
function backToMenu(){ document.getElementById('menu').style.display='block'; document.getElementById('game').style.display='none'; document.getElementById('levelSelect').style.display='none'; document.getElementById('scores').style.display='none'; }
document.getElementById('menu').style.display='block';
</script>
</body>
</html>
