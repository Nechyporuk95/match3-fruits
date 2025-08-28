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
#menu, #game, #levelSelect {
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
  transition: transform 0.3s ease, top 0.3s ease;
  position: relative;
}
.cell.match {
  transform: scale(1.5);
  opacity: 0;
}
#score, #task {
  font-size:20px;
  margin:10px;
}
</style>
</head>
<body>
<div id="menu">
<h1>Фрукты 3 в ряд</h1>
<button class="btn" onclick="startGame()">Начать игру</button>
<button class="btn" onclick="showLevelSelect()">Выбрать уровень</button>
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
<audio id="matchSound" src="https://actions.google.com/sounds/v1/cartoon/wood_plank_flicks.ogg"></audio>
<script>
const fruits = ["🍎","🍌","🍇","🍊","🍉","🍒","🍓","🍍","🍏","🥝"];
let board=[];
let size=6;
let score=0;
let selected=null;
let currentLevel=1;
const boardEl = document.getElementById("board");
const scoreEl = document.getElementById("score");
const taskEl = document.getElementById("task");
const matchSound = document.getElementById("matchSound");

// Генерация 50 уровней с уникальными заданиями и разными полями
const LEVELS=[];
for(let i=1;i<=50;i++){
  let sz=6+Math.floor(i/10); // увеличиваем размер поля каждые 10 уровней
  let type='points';
  let fruit=null;
  if(i%5===2) { type='collect'; fruit=fruits[Math.floor(Math.random()*8)]; }
  else if(i%5===3) { type='bonus'; fruit=i%2===0?'🍏':'🥝'; }
  else if(i%5===4) { type='collect'; fruit=fruits[Math.floor(Math.random()*8)]; }
  LEVELS.push({size:sz, task:{type:type, fruit:fruit, amount:100+i*10}});
}

function randomFruit(){
  let r=Math.random();
  if(r<0.05) return "🍏";
  if(r<0.1) return "🥝";
  return fruits[Math.floor(Math.random()*8)];
}

function initBoard(){
  size=LEVELS[currentLevel-1].size;
  boardEl.style.gridTemplateColumns=`repeat(${size},50px)`;
  boardEl.style.gridTemplateRows=`repeat(${size},50px)`;
  board=[];
  boardEl.innerHTML='';
  for(let r=0;r<size;r++){
    let row=[];
    for(let c=0;c<size;c++){
      let f=randomFruit();
      row.push(f);
      const cell=document.createElement("div");
      cell.classList.add("cell");
      cell.textContent=f;
      cell.dataset.row=r;
      cell.dataset.col=c;
      cell.style.top='-60px';
      boardEl.appendChild(cell);
      setTimeout(()=>cell.style.top='0px', r*50+10*c);
      cell.onclick = ()=>selectCell(r,c,cell);
    }
    board.push(row);
  }
  score=0;
  scoreEl.textContent="Очки: 0";
  showTask();
}

function showTask(){
  const t=LEVELS[currentLevel-1].task;
  if(t.type==='points') taskEl.textContent=`Задание: набрать ${t.amount} очков`;
  else if(t.type==='collect') taskEl.textContent=`Задание: собрать ${t.amount} ${t.fruit}`;
  else if(t.type==='bonus') taskEl.textContent=`Задание: активировать ${t.amount} ${t.fruit}`;
}

function selectCell(r,c,cell){
  if(!selected){selected={r,c,cell}; cell.style.transform='scale(1.2)';}
  else{
    let sr=selected.r, sc=selected.c;
    if(Math.abs(sr-r)+Math.abs(sc-c)===1){ swap(sr,sc,r,c); if(!checkMatches()){ swap(sr,sc,r,c); } }
    selected.cell.style.transform='scale(1)';
    selected=null;
  }
}

function swap(r1,c1,r2,c2){
  let temp=board[r1][c1]; board[r1][c1]=board[r2][c2]; board[r2][c2]=temp;
  renderBoard();
}

function checkMatches(){
  let matched=[];
  for(let r=0;r<size;r++){ for(let c=0;c<size-2;c++){ let f=board[r][c]; if(f&&f===board[r][c+1]&&f===board[r][c+2]) matched.push([r,c],[r,c+1],[r,c+2]); } }
  for(let c=0;c<size;c++){ for(let r=0;r<size-2;r++){ let f=board[r][c]; if(f&&f===board[r+1][c]&&f===board[r+2][c]) matched.push([r,c],[r+1,c],[r+2,c]); } }
  if(matched.length>0){
    matched.forEach(([r,c])=>{
      let f=board[r][c];
      if(f==='🍏'){ for(let col=0;col<size;col++) matched.push([r,col]); score+=100; }
      if(f==='🥝'){ for(let row=0;row<size;row++) matched.push([row,c]); score+=100; }
    });
    matched.forEach(([r,c])=>{ const idx=r*size+c; const cell=boardEl.children[idx]; cell.classList.add('match'); });
    setTimeout(()=>{ matched.forEach(([r,c])=>board[r][c]=null); dropFruits(); renderBoard(); checkTask(); },300);
    score+=matched.length*10;
    scoreEl.textContent="Очки: "+score;
    matchSound.play();
    return true;
  }
  return false;
}

function dropFruits(){
  for(let c=0;c<size;c++){
    let empty=0;
    for(let r=size-1;r>=0;r--){ if(!board[r][c]) empty++; else if(empty>0){ board[r+empty][c]=board[r][c]; board[r][c]=null; } }
    for(let r=0;r<empty;r++){ board[r][c]=randomFruit(); }
  }
}

function renderBoard(){
  for(let r=0;r<size;r++){ for(let c=0;c<size;c++){ const idx=r*size+c; const cell=boardEl.children[idx]; cell.textContent=board[r][c]; cell.classList.remove('match'); cell.style.transform='scale(1)'; cell.style.top='0px'; } }
}

function checkTask(){
  const t=LEVELS[currentLevel-1].task;
  let done=false;
  if(t.type==='points'&&score>=t.amount) done=true;
  // можно добавить подсчёт collect и bonus по фруктам
  if(done){
    alert('Уровень пройден!');
    currentLevel++; if(currentLevel>LEVELS.length) currentLevel=1;
    initBoard();
  }
}

function startGame(){
  document.getElementById('menu').style.display='none';
  document.getElementById('levelSelect').style.display='none';
  document.getElementById('game').style.display='block';
  initBoard();
}

function showLevelSelect(){
  document.getElementById('menu').style.display='none';
  document.getElementById('levelSelect').style.display='block';
  const levelsDiv=document.getElementById('levels'); levelsDiv.innerHTML='';
  for(let i=1;i<=50;i++){ const btn=document.createElement('button'); btn.className='btn'; btn.textContent='Уровень '+i; btn.onclick=()=>{currentLevel=i; initBoard(); document.getElementById('levelSelect').style.display='none'; document.getElementById('game').style.display='block';}; levelsDiv.appendChild(btn); }
}

function backToMenu(){ document.getElementById('menu').style.display='block'; document.getElementById('game').style.display='none'; document.getElementById('levelSelect').style.display='none'; }

document.getElementById('menu').style.display='block';
</script>
</body>
</html>
