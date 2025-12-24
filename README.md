# -const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// =====================
// 플레이어 (부드러운 이동)
// =====================
const PLAYER_SIZE = 50;
const PLAYER_SPEED = 260; // px/s (원하면 200~350 사이 조절)

let playerX = canvas.width / 2 - PLAYER_SIZE / 2;
let playerY = canvas.height / 2 - PLAYER_SIZE / 2;

const keys = { ArrowUp: false, ArrowDown: false, ArrowLeft: false, ArrowRight: false };
let lastFrameTime = performance.now();

// =====================
// 게임 상태
// =====================
let isGameOver = false;
let isInvincible = false;
let isExam = false;
let examClearedOnce = false;

let startTime = Date.now();
let currentTime = Date.now();
let examStartTime = 0;

let itemCount = 0;

// ✅ OMR(=부활권) 최대 1개
let bonusItemCount = 0; // 0 or 1

let highScore = Number(localStorage.getItem('highScore')) || 0;

// ✅ 10초 단위 시험 확률이 프레임마다 굴려지는 것 방지
let lastExamRollSecond = -1;

// =====================
// 시험 타입 설정 (불/물)
// =====================
let examType = 'normal'; // 'fire' | 'water'
const EXAM_DURATION = 20;      // 시험 시간(초)
const EXAM_BASE_SPEED = 1.35;  // ⭐ 기본 시험 속도(조금 빠르게)

const FIRE_CHANCE = 0.5;       // 불시험 확률(나머지는 물시험)
const FIRE_ENEMY_COUNT = 1;
const FIRE_SPEED_MULT = 2.1;   // 불시험: 매우 빠름

const WATER_ENEMY_COUNT = 5;
const WATER_SPEED_MULT = 0.75; // 물시험: 느리지만 많음

// =====================
// 이미지
// =====================
const background = new Image();
background.src = '교실.png';

const playerImg = new Image();
playerImg.src = '학생.png';

const enemyImg = new Image();
enemyImg.src = 'KSA.png';

const itemImg = new Image();
itemImg.src = '대수.png';

const omrImg = new Image();
omrImg.src = 'omr.png';

// =====================
// 적 & 아이템
// =====================
const enemies = [];

let item = {
  x: Math.random() * (canvas.width - 40),
  y: Math.random() * (canvas.height - 40),
  width: 40,
  height: 40,
  active: true
};

// 시험 중 5초 남았을 때 나오는 OMR 카드
let omr = {
  x: 0,
  y: 0,
  width: 40,
  height: 40,
  active: false
};
let omrSpawnedThisExam = false;

// =====================
// 유틸
// =====================
function clamp(v, min, max) {
  return Math.max(min, Math.min(max, v));
}

// =====================
// 적 초기화
// =====================
function initializeEnemies(speedMultiplier = 1, count = 3) {
  enemies.length = 0;

  for (let i = 0; i < count; i++) {
    enemies.push({
      x: Math.random() * (canvas.width - 50),
      y: Math.random() * (canvas.height - 50),
      width: 50,
      height: 50,
      speedX: (Math.random() * 2 + 1) * speedMultiplier,
      speedY: (Math.random() * 2 + 1) * speedMultiplier
    });
  }
}

// =====================
// 그리기
// =====================
function drawBackground() {
  ctx.drawImage(background, 0, 0, canvas.width, canvas.height);
}

function drawPlayer() {
  ctx.drawImage(playerImg, playerX, playerY, PLAYER_SIZE, PLAYER_SIZE);
}

function drawEnemies() {
  enemies.forEach(e => ctx.drawImage(enemyImg, e.x, e.y, e.width, e.height));
}

function drawItem() {
  if (item.active) ctx.drawImage(itemImg, item.x, item.y, item.width, item.height);
}

function drawOMR() {
  if (omr.active) ctx.drawImage(omrImg, omr.x, omr.y, omr.width, omr.height);
}

// =====================
// 이동
// =====================
function updatePlayer(dt) {
  let dx = 0, dy = 0;
  if (keys.ArrowRight) dx += 1;
  if (keys.ArrowLeft)  dx -= 1;
  if (keys.ArrowDown)  dy += 1;
  if (keys.ArrowUp)    dy -= 1;

  // 대각선 속도 보정
  if (dx !== 0 && dy !== 0) {
    const inv = 1 / Math.sqrt(2);
    dx *= inv;
    dy *= inv;
  }

  playerX += dx * PLAYER_SPEED * dt;
  playerY += dy * PLAYER_SPEED * dt;

  playerX = clamp(playerX, 0, canvas.width - PLAYER_SIZE);
  playerY = clamp(playerY, 0, canvas.height - PLAYER_SIZE);
}

function moveEnemies() {
  enemies.forEach(e => {
    e.x += e.speedX;
    e.y += e.speedY;

    if (e.x < 0 || e.x + e.width > canvas.width) e.speedX *= -1;
    if (e.y < 0 || e.y + e.height > canvas.height) e.speedY *= -1;
  });
}

// =====================
// 리스폰
// =====================
function respawnItem() {
  item.x = Math.random() * (canvas.width - item.width);
  item.y = Math.random() * (canvas.height - item.height);
}

function respawnOMR() {
  omr.x = Math.random() * (canvas.width - omr.width);
  omr.y = Math.random() * (canvas.height - omr.height);
}

// =====================
// 충돌
// =====================
function getPlayerHitbox() {
  // 플레이어 이미지(50x50) 기준으로 중앙 쪽만 판정
  return {
    x: playerX + 10,
    y: playerY + 10,
    width: 30,
    height: 30
  };
}

function checkEnemyCollision() {
  if (isInvincible) return;

  const p = getPlayerHitbox();

  for (const e of enemies) {
    if (
      p.x < e.x + e.width &&
      p.x + p.width > e.x &&
      p.y < e.y + e.height &&
      p.y + p.height > e.y
    ) {
      // ✅ OMR(부활권) 있으면 1번 부활
      if (bonusItemCount > 0) {
        bonusItemCount = 0;
        isInvincible = true;

        // 부활: 중앙으로 이동
        playerX = canvas.width / 2 - PLAYER_SIZE / 2;
        playerY = canvas.height / 2 - PLAYER_SIZE / 2;

        setTimeout(() => { isInvincible = false; }, 2000);
        return;
      }

      isGameOver = true;
      return;
    }
  }
}

function checkItemCollision() {
  if (!item.active) return;

  const p = getPlayerHitbox();

  if (
    p.x < item.x + item.width &&
    p.x + p.width > item.x &&
    p.y < item.y + item.height &&
    p.y + p.height > item.y
  ) {
    startTime -= 10000; // +10초
    itemCount++;
    respawnItem();

    // ✅ 아이템 먹을 때 시험 시작 확률 5%
    if (examClearedOnce && !isExam && Math.random() < 0.05) {
      startExam();
    }
  }
}

function checkOMRCollision() {
  if (!omr.active) return;

  const p = getPlayerHitbox();

  if (
    p.x < omr.x + omr.width &&
    p.x + p.width > omr.x &&
    p.y < omr.y + omr.height &&
    p.y + p.height > omr.y
  ) {
    omr.active = false;

    if (bonusItemCount < 1) {
      bonusItemCount = 1;
      alert('📝 OMR 획득! 부활권 1개 지급!');
    } else {
      alert('📝 OMR 획득! (이미 부활권 1개 보유 중)');
    }
  }
}

// =====================
// 시험 로직 (불시험/물시험)
// =====================
function pickExamType() {
  examType = Math.random() < FIRE_CHANCE ? 'fire' : 'water';
}

function startExam() {
  if (isExam) return;

  pickExamType();

  isExam = true;
  isInvincible = true;
  examStartTime = Date.now();

  item.active = false;

  // 시험 시작 시 OMR 리셋
  omr.active = false;
  omrSpawnedThisExam = false;

  // 타입별 적 세팅
  if (examType === 'fire') {
    initializeEnemies(EXAM_BASE_SPEED * FIRE_SPEED_MULT, FIRE_ENEMY_COUNT);
  } else {
    initializeEnemies(EXAM_BASE_SPEED * WATER_SPEED_MULT, WATER_ENEMY_COUNT);
  }

  // 시험 시작 3초 무적
  setTimeout(() => { isInvincible = false; }, 3000);
}

function updateExam() {
  if (!isExam) return;

  const examElapsed = (Date.now() - examStartTime) / 1000;
  const remain = Math.max(0, EXAM_DURATION - Math.floor(examElapsed));

  // ✅ 5초 남았을 때 랜덤 OMR 등장 (부활권 없을 때만)
  if (!omrSpawnedThisExam && remain <= 5) {
    omrSpawnedThisExam = true;

    // 불시험이 더 빡세니까 OMR 확률 약간↑
    let spawnChance = 0.5;
    if (examType === 'fire') spawnChance = 0.65;
    if (examType === 'water') spawnChance = 0.45;

    if (bonusItemCount === 0 && Math.random() < spawnChance) {
      respawnOMR();
      omr.active = true;
    }
  }

  // 시험 종료
  if (examElapsed >= EXAM_DURATION) {
    const reward = 200000; // 시간 보상
    startTime -= reward;

    isExam = false;
    examClearedOnce = true;

    item.active = true;
    respawnItem();

    omr.active = false;

    // 일반 상태 복귀
    initializeEnemies(1, 3);
  }
}

// =====================
// UI
// =====================
function drawTime() {
  currentTime = Date.now();
  const elapsed = (currentTime - startTime) / 1000;

  ctx.fillStyle = 'black';
  ctx.font = '20px Arial';
  ctx.fillText(`Time: ${elapsed.toFixed(2)}`, 10, 30);
  ctx.fillText(`Items: ${itemCount}`, 10, 55);
  ctx.fillText(`부활권(OMR): ${bonusItemCount}/1`, 10, 80);
  ctx.fillText(`High: ${highScore.toFixed(2)}`, 10, 105);

  // 최초 시험 조건
  if (!isExam && !examClearedOnce) {
    if (itemCount >= 10 || elapsed >= 200) startExam();
  }

  // ✅ 이후 시험 확률: 10초마다 15% (딱 1번만 굴리기)
  const sec = Math.floor(elapsed);
  if (examClearedOnce && !isExam && sec % 10 === 0 && sec !== lastExamRollSecond) {
    lastExamRollSecond = sec;
    if (Math.random() < 0.15) startExam();
  }
}

function drawExamText() {
  if (!isExam) return;

  const remain = Math.max(
    0,
    EXAM_DURATION - Math.floor((Date.now() - examStartTime) / 1000)
  );

  let title = '시험 시작';
  let color = 'red';
  if (examType === 'fire') { title = '🔥 불시험 시작'; color = 'red'; }
  if (examType === 'water') { title = '💧 물시험 시작'; color = 'blue'; }

  ctx.fillStyle = color;
  ctx.font = '40px Arial';
  ctx.textAlign = 'center';
  ctx.fillText(title, canvas.width / 2, canvas.height / 2 - 30);
  ctx.fillText(`남은 시간: ${remain}s`, canvas.width / 2, canvas.height / 2 + 20);
  ctx.textAlign = 'left';
}

// =====================
function updateHighScore() {
  const elapsed = (Date.now() - startTime) / 1000;
  if (elapsed > highScore) {
    highScore = elapsed;
    localStorage.setItem('highScore', highScore);
  }
}

// =====================
// 메인 루프
// =====================
function gameLoop(now) {
  const dt = Math.min(0.033, (now - lastFrameTime) / 1000); // 렉 방지(최대 33ms)
  lastFrameTime = now;

  if (isGameOver) {
    updateHighScore();
    alert('Game Over');
    return;
  }

  // ✅ 부드러운 이동
  updatePlayer(dt);

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  drawBackground();
  drawPlayer();
  drawEnemies();
  drawItem();
  drawOMR();

  moveEnemies();

  checkEnemyCollision();
  checkItemCollision();
  checkOMRCollision();

  updateExam();
  drawTime();
  drawExamText();

  requestAnimationFrame(gameLoop);
}

// =====================
// 입력 (키 누르고 있는 동안 이동)
// =====================
window.addEventListener('keydown', e => {
  if (e.key in keys) {
    keys[e.key] = true;
    e.preventDefault();
  }
});

window.addEventListener('keyup', e => {
  if (e.key in keys) {
    keys[e.key] = false;
    e.preventDefault();
  }
});

// 시작
initializeEnemies(1, 3);
requestAnimationFrame(gameLoop);
