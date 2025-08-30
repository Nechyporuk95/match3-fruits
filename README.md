<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
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
  min-height:100vh;
}
#menu, #game, #levelSelect, #scores {
  display:none;
  text-align:center;
  background: rgba(255,255,255,0.2);
  padding: 15px;
  border-radius: 15px;
  width:90%;
  max-width:400px;
}
.btn {
  background: #ff7f50;
  border: none;
  padding: 10px 15px;
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
  gap: 3px;
  margin: 15px auto;
  justify-content: center;
}
.cell {
  display:flex;
  align-items:center;
  justify-content:center;
  font-size: 28px;
  cursor:pointer;
  user-select:none;
  position: relative;
  border-radius: 12px;
  background: rgba(255,255,255,0.4);
  box-shadow: 0 3px 10px rgba(0,0,0,0.25);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}
.cell.matchGlow {
  animation: glow 0.6s infinite alternate;
}
@keyframes glow {
  0% { box-shadow: 0 0 10px rgba(255,255,255,0.7), 0 0 20px rgba(255,255,255,0.5); }
  100% { box-shadow: 0 0 20px rgba(255,255,255,1), 0 0 30px rgba(255,255,255,0.7); }
}
.cell.remove {
  animation: explode 0.4s forwards;
}
@keyframes explode {
  0% { transform: scale(1) rotate(0deg); opacity:1; }
  50% { transform: scale(1.6) rotate(180deg); opacity:0.7; }
  100% { transform: scale(0) rotate(360deg); opacity:0; }
}
#score, #task {
  font-size:18px;
  margin:8px;
}
.cell.hint { animation: hintBlink 0.8s infinite; }
@keyframes hintBlink {
  0% { transform: scale(1); }
  50% { transform: scale(1.3); }
  100% { transform: scale(1); }
}
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
let unlockedLevels = 1;
const boardEl=document.getElementById("board");
const scoreEl=document.getElementById("score");
const taskEl=document.getElementById("task");
const matchSound=document.getElementById("matchSound");

// создание уровней
const LEVELS=[];
for(let i=1;i<=50;i++){
  let sz=6+Math.floor(i/10);
  let type='points', fruit=null, amount=100+i*10;
  if(i%5===2){ type='collect'; fruit=fruits[Math.floor(Math.random()*fruits.length)]; amount=20+Math.floor(i/2); }
  else if(i%5===3){ type='bonus'; fruit=i%2===0?'🍏':'🥝'; amount=5+Math.floor(i/3); }
  else if(i%5===4){ type='collect'; fruit=fruits[Math.floor(Math.random()*fruits.length)]; amount=25+Math.floor(i/2); }
  LEVELS.push({size:sz, task:{type, fruit, amount}});
}

// рандомизация фрукта
function randomFruit(){
  const t=LEVELS[currentLevel-1].task;
  let r=Math.random();
  if(r<0.05) return "🍏";
  if(r<0.1) return "🥝";
  if(t.type==='collect' && Math.random()<0.25) return t.fruit;
  return fruits[Math.floor(Math.random()*fruits.length)];
}

// Инициализация поля
function initBoard(){
  size=LEVELS[currentLevel-1].size;
  boardEl.style.gridTemplateColumns=`repeat(${size},50px)`;
  boardEl.style.gridTemplateRows=`repeat(${size},50px)`;
  do{
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
  }while(findMoves().length===0);
  score=0; collected=0;
  scoreEl.textContent="Очки: 0"; showTask(); resetIdleTimer();
  resolveBoard();
}

// Показ задания
function showTask(){
  const t=LEVELS[currentLevel-1].task;
  if(t.type==='points') taskEl.textContent=`Задание: набрать ${t.amount} очков`;
  else taskEl.textContent=`Задание: ${t.type==='collect'?'собрать':'активировать'} ${t.amount} ${t.fruit} (собрано: ${collected}/${t.amount})`;
}

// Выбор клетки
function selectCell(r,c,cell){
  if(!selected){selected={r,c,cell}; cell.style.transform='scale(1.2)';}
  else{
    let sr=selected.r, sc=selected.c;
    if(Math.abs(sr-r)+Math.abs(sc-c)===1){ swap(sr,sc,r,c); resolveBoard(); }
    selected.cell.style.transform='scale(1)'; selected=null;
  }
}

// swap с визуальным обновлением
function swap(r1,c1,r2,c2){ [board[r1][c1],board[r2][c2]]=[board[r2][c2],board[r1][c1]]; renderBoard(); }

// Находим совпадения
function findMatches(){
  let matches=[];
  for(let r=0;r<size;r++)for(let c=0;c<size-2;c++){
    let f=board[r][c];
    if(f && f===board[r][c+1] && f===board[r][c+2]){
      let group=[[r,c],[r,c+1],[r,c+2]]; let cc=c+3; while(cc<size && board[r][cc]===f){ group.push([r,cc]); cc++; }
      matches.push(...group);
    }
  }
  for(let c=0;c<size;c++)for(let r=0;r<size-2;r++){
    let f=board[r][c];
    if(f && f===board[r+1][c] && f===board[r+2][c]){
      let group=[[r,c],[r+1,c],[r+2,c]]; let rr=r+3; while(rr<size && board[rr][c]===f){ group.push([rr,c]); rr++; }
      matches.push(...group);
    }
  }
  return [...new Set(matches.map(x=>x.toString()))].map(x=>x.split(',').map(Number));
}

// Подсветка совпадений
function highlightMatches(matches){ clearGlow(); matches.forEach(([r,c])=>boardEl.children[r*size+c].classList.add('matchGlow')); }
function clearGlow(){ [...boardEl.children].forEach(cell=>cell.classList.remove('matchGlow')); }

// Удаление совпадений
function removeMatches(matches){
  highlightMatches(matches);
  matches.forEach(([r,c])=>{
    const cell=boardEl.children[r*size+c];
    cell.classList.add("remove");
    board[r][c]=null;
    score+=10; collected++;
  });
  scoreEl.textContent="Очки: "+score;
  showTask();
}

// Анимация падения фруктов
function dropFruits(){
  for(let c=0;c<size;c++){
    let empty=[];
    for(let r=size-1;r>=0;r--){
      if(!board[r][c]) empty.push(r);
      else if(empty.length>0){
        let newRow=empty.shift();
        board[newRow][c]=board[r][c]; board[r][c]=null; empty.push(r);
        const cell=boardEl.children[newRow*size+c];
        cell.style.transform='translateY(-60px) rotate(-15deg)';
        setTimeout(()=>{cell.style.transition='transform 0.4s ease'; cell.style.transform='translateY(0) rotate(0deg)';},50);
      }
    }
    for(let e of empty) board[e][c]=randomFruit();
  }
  renderBoard();
}

// Отображение поля
function renderBoard(){ for(let r=0;r<size;r++)for(let c=0;c<size;c++){ const cell=boardEl.children[r*size+c]; cell.textContent=board[r][c]; cell.classList.remove('remove','hint'); cell.style.transform='scale(1)'; } }

// Решение совпадений
function resolveBoard(){
  let matches=findMatches();
  if(matches.length>0){
    removeMatches(matches);
    matchSound.play();
    setTimeout(()=>{ dropFruits(); resolveBoard(); checkTask(); },400);
  } else if(findMoves().length===0){ reshuffle(); }
}

// Проверка задания
function checkTask(){
  const t=LEVELS[currentLevel-1].task;
  if((t.type==='points' && score>=t.amount) || (t.type!=='points' && collected>=t.amount)){
    saveScore(score); alert(`Уровень ${currentLevel} пройден!`);
    if(currentLevel>=unlockedLevels && unlockedLevels<LEVELS.length) unlockedLevels++;
    currentLevel++; if(currentLevel>LEVELS.length) currentLevel=LEVELS.length;
    initBoard(); updateLevelButtons();
  }
}

// Находим возможные ходы
function findMoves(){
  let m=[];
  function swapTmp(r1,c1,r2,c2){ [board[r1][c1],board[r2][c2]]=[board[r2][c2],board[r1][c1]]; }
  for(let r=0;r<size;r++)for(let c=0;c<size;c++){
    if(c<size-1){ swapTmp(r,c,r,c+1); if(findMatches().length>0)m.push([[r,c],[r,c+1]]); swapTmp(r,c,r,c+1);}
    if(r<size-1){ swapTmp(r,c,r+1,c); if(findMatches().length>0)m.push([[r,c],[r+1,c]]); swapTmp(r,c,r+1,c);}
  }
  return m;
}

// Если ходов нет — перемешиваем поле
function reshuffle(){
  let flat=board.flat().filter(f=>f); flat.sort(()=>Math.random()-0.5);
  for(let r=0;r<size;r++) for(let c=0;c<size;c++) board[r][c]=flat[r*size+c];
  renderBoard();
  if(findMoves().length===0) reshuffle();
}

// Подсказки
function resetIdleTimer(){ clearTimeout(idleTimer); clearHint(); idleTimer=setTimeout(showHint,5000); }
function showHint(){ let moves=findMoves(); if(moves.length>0){ let move=moves[Math.floor(Math.random()*moves.length)]; move.forEach(([r,c])=>boardEl.children[r*size+c].classList.add("hint")); } }
function clearHint(){ [...boardEl.children].forEach(cell=>cell.classList.remove("hint")); }

// Рекорды
function saveScore(s){ let scores=JSON.parse(localStorage.getItem("scores")||"[]"); scores.push(s); scores.sort((a,b)=>b-a); scores=scores.slice(0,5); localStorage.setItem("scores",JSON.stringify(scores)); }
function showScores(){ document.getElementById("menu").style.display='none'; document.getElementById("scores").style.display='block'; let scores=JSON.parse(localStorage.getItem("scores")||"[]"); let tbody=document.getElementById("scoreTable"); tbody.innerHTML=''; scores.forEach((s,i)=>{ tbody.innerHTML+=`<tr><td>${i+1}</td><td>${s}</td></tr>`; }); }

// Меню уровней
function startGame(){ document.getElementById('menu').style.display='none'; document.getElementById('levelSelect').style.display='none'; document.getElementById('scores').style.display='none'; document.getElementById('game').style.display='block'; initBoard(); }
function showLevelSelect(){ document.getElementById('menu').style.display='none'; document.getElementById('levelSelect').style.display='block'; updateLevelButtons(); }
function updateLevelButtons(){ const levelsDiv=document.getElementById('levels'); levelsDiv.innerHTML=''; for(let i=1;i<=50;i++){ const btn=document.createElement('button'); btn.className='btn'; btn.textContent='Уровень '+i; if(i<=unlockedLevels){ btn.onclick=()=>{currentLevel=i; initBoard(); document.getElementById('levelSelect').style.display='none'; document.getElementById('game').style.display='block';}; } else { btn.disabled=true; btn.style.background='#999'; } levelsDiv.appendChild(btn);} }
function backToMenu(){ document.getElementById('menu').style.display='block'; document.getElementById('game').style.display='none'; document.getElementById('levelSelect').style.display='none'; document.getElementById('scores').style.display='none'; }

document.getElementById('menu').style.display='block';

// --- Свайпы для мобильных ---
let touchStart=null;
boardEl.addEventListener('touchstart', e=>{ if(e.touches.length===1){ touchStart={x:e.touches[0].clientX,y:e.touches[0].clientY}; }});
boardEl.addEventListener('touchend', e=>{
  if(!touchStart) return;
  let dx=e.changedTouches[0].clientX - touchStart.x;
  let dy=e.changedTouches[0].clientY - touchStart.y;
  let absX=Math.abs(dx), absY=Math.abs(dy);
  if(absX>30 || absY>30){
    let r=selected?.r, c=selected?.c;
    if(r==null || c==null) { touchStart=null; return; }
    if(absX>absY){ if(dx>0 && c<size-1) swap(r,c,r,c+1); else if(dx<0 && c>0) swap(r,c,r,c-1); }
    else { if(dy>0 && r<size-1) swap(r,c,r+1,c); else if(dy<0 && r>0) swap(r,c,r-1,c); }
    resolveBoard();
    selected.cell.style.transform='scale(1)';
    selected=null;
  }
  touchStart=null;
});
</script>
</body>
</html>
