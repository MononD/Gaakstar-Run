# Gaakstar-Run
Gaakstar Run - Chrome Dino style game
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Gaakstar Run</title>
  <style>
    html,body{margin:0;height:100%;background:#f7f7f7;font-family:system-ui,-apple-system,Segoe UI,Roboto,Apple SD Gothic Neo,Noto Sans KR,sans-serif;}
    .wrap{display:flex;align-items:center;justify-content:center;height:100%;}
    canvas{background:#fff; border:1px solid #ddd; image-rendering:auto; touch-action:manipulation;}
    .hint{position:fixed;left:0;right:0;bottom:14px;text-align:center;color:#666;font-size:14px;user-select:none;}
  </style>
</head>
<body>
<div class="wrap">
  <canvas id="c" width="900" height="240" aria-label="Gaakstar Run"></canvas>
</div>
<div class="hint">점프: Space / ↑ / 터치 · 재시작: R</div>

<script>
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');

  // ====== CONFIG ======
  const GROUND_Y = 190;          // 바닥선
  const GRAVITY = 0.9;
  const JUMP_V = -14.5;
  const BASE_SPEED = 6.2;        // 시작 속도
  const SPEED_UP = 0.0009;       // 시간에 따라 빨라짐
  const OBST_MIN = 380;          // 장애물 최소 간격
  const OBST_MAX = 760;          // 장애물 최대 간격
  const BIRD_CHANCE = 0.22;      // 새 등장 확률
  const NIGHT_EVERY = 900;       // 점수 기준(대략) 밤 전환 주기

  // ====== STATE ======
  let running = false;
  let gameOver = false;
  let t = 0;
  let score = 0;
  let best = Number(localStorage.getItem('gaakstar_best') || 0);
  let speed = BASE_SPEED;
  let nextObstX = canvas.width + 300;

  const rand = (a,b)=>a+Math.random()*(b-a);
  const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));

  // ====== PLAYER IMAGE ======
  const playerImg = new Image();
  playerImg.src = 'player.png'; // <- 여기에 네 사진 파일명을 맞춰줘
  let playerImgReady = false;
  playerImg.onload = () => { playerImgReady = true; };
  playerImg.onerror = () => { playerImgReady = false; };

  const player = {
    x: 90,
    y: GROUND_Y,
    w: 52,
    h: 72,
    vy: 0,
    onGround: true,
    duck: false,
  };

  const obstacles = []; // {x,y,w,h,type}

  function reset() {
    running = false;
    gameOver = false;
    t = 0;
    score = 0;
    speed = BASE_SPEED;
    nextObstX = canvas.width + 300;
    obstacles.length = 0;
    player.y = GROUND_Y;
    player.vy = 0;
    player.onGround = true;
    player.duck = false;
  }

  function start() {
    if (gameOver) return;
    running = true;
  }

  function spawnObstacle() {
    const isBird = Math.random() < BIRD_CHANCE && score > 250;
    if (isBird) {
      const heights = [GROUND_Y - 95, GROUND_Y - 65, GROUND_Y - 45];
      const y = heights[Math.floor(Math.random()*heights.length)];
      obstacles.push({ x: nextObstX, y, w: 46, h: 30, type: 'bird' });
    } else {
      // cactus cluster variety
      const sizes = [
        {w:22,h:46},{w:26,h:52},{w:34,h:60},{w:44,h:62}
      ];
      const s = sizes[Math.floor(Math.random()*sizes.length)];
      obstacles.push({ x: nextObstX, y: GROUND_Y - s.h, w: s.w, h: s.h, type: 'cactus' });
    }
    nextObstX += rand(OBST_MIN, OBST_MAX);
  }

  function aabb(ax,ay,aw,ah, bx,by,bw,bh) {
    return ax < bx + bw && ax + aw > bx && ay < by + bh && ay + ah > by;
  }

  function drawBackground() {
    const isNight = Math.floor(score / NIGHT_EVERY) % 2 === 1;
    ctx.fillStyle = isNight ? '#1c1f2a' : '#ffffff';
    ctx.fillRect(0,0,canvas.width,canvas.height);

    // horizon line
    ctx.strokeStyle = isNight ? '#3a3f55' : '#e5e5e5';
    ctx.beginPath();
    ctx.moveTo(0, GROUND_Y+1);
    ctx.lineTo(canvas.width, GROUND_Y+1);
    ctx.stroke();

    // 작은 구름/별 느낌 점들
    ctx.fillStyle = isNight ? 'rgba(255,255,255,0.65)' : 'rgba(0,0,0,0.12)';
    for (let i=0;i<10;i++){
      const x = (i*110 + (t*0.2)) % (canvas.width+120) - 60;
      const y = isNight ? (40 + (i%3)*18) : (38 + (i%3)*14);
      ctx.beginPath(); ctx.arc(x,y, isNight?1.6:1.2, 0, Math.PI*2); ctx.fill();
    }

    // 바닥 장식(점선)
    ctx.strokeStyle = isNight ? 'rgba(255,255,255,0.10)' : 'rgba(0,0,0,0.08)';
    ctx.beginPath();
    for(let x=0;x<canvas.width;x+=24){
      ctx.moveTo(x, GROUND_Y+14);
      ctx.lineTo(x+10, GROUND_Y+14);
    }
    ctx.stroke();

    // Title UI
    ctx.fillStyle = isNight ? '#ffffff' : '#111';
    ctx.font = '600 18px system-ui, sans-serif';
    ctx.fillText('Gaakstar Run', 18, 28);

    // Score UI
    ctx.font = '14px system-ui, sans-serif';
    ctx.fillStyle = isNight ? 'rgba(255,255,255,0.85)' : 'rgba(0,0,0,0.75)';
    const s = String(Math.floor(score)).padStart(5,'0');
    const b = String(Math.floor(best)).padStart(5,'0');
    ctx.fillText(`SCORE ${s}   BEST ${b}`, canvas.width - 230, 26);
  }

  function drawPlayer() {
    const isNight = Math.floor(score / NIGHT_EVERY) % 2 === 1;

    // hitbox(간단 튜닝: 사진 캐릭터가 좀 커도 판정은 살짝 작게)
    const hb = getPlayerHitbox();

    if (playerImgReady) {
      // 사진 그대로 그리기
      // (사진 크기에 따라 자동 스케일)
      const targetW = player.duck ? 70 : 58;
      const targetH = player.duck ? 44 : 78;
      const x = player.x;
      const y = player.y - targetH;

      ctx.save();
      // 살짝 그림자
      ctx.globalAlpha = isNight ? 0.25 : 0.18;
      ctx.fillStyle = '#000';
      ctx.beginPath();
      ctx.ellipse(x+targetW*0.5, GROUND_Y+10, targetW*0.35, 6, 0, 0, Math.PI*2);
      ctx.fill();
      ctx.globalAlpha = 1;

      ctx.drawImage(playerImg, x, y, targetW, targetH);
      ctx.restore();
    } else {
      // 이미지 없을 때 임시 플레이어
      ctx.fillStyle = isNight ? '#fff' : '#111';
      ctx.fillRect(hb.x, hb.y, hb.w, hb.h);
    }
  }

  function getPlayerHitbox(){
    const w = player.duck ? 48 : 42;
    const h = player.duck ? 36 : 62;
    const x = player.x + 8;
    const y = player.y - h;
    return {x,y,w,h};
  }

  function drawObstacle(o) {
    const isNight = Math.floor(score / NIGHT_EVERY) % 2 === 1;
    ctx.fillStyle = isNight ? '#ffffff' : '#111';

    if (o.type === 'bird') {
      // 간단한 새(블럭) 표현
      ctx.fillRect(o.x, o.y, o.w, o.h);
      // 날개 느낌
      ctx.fillRect(o.x+6, o.y-10, 12, 8);
      ctx.fillRect(o.x+26, o.y-8, 12, 6);
    } else {
      // cactus
      ctx.fillRect(o.x, o.y, o.w, o.h);
      // 팔 같은 느낌
      if (o.w >= 30) {
        ctx.fillRect(o.x+6, o.y+12, 10, 16);
        ctx.fillRect(o.x+o.w-16, o.y+18, 10, 16);
      }
    }
  }

  function drawStartOverlay() {
    const isNight = Math.floor(score / NIGHT_EVERY) % 2 === 1;
    if (!running && !gameOver) {
      ctx.fillStyle = isNight ? 'rgba(0,0,0,0.35)' : 'rgba(0,0,0,0.06)';
      ctx.fillRect(0,0,canvas.width,canvas.height);
      ctx.fillStyle = isNight ? '#fff' : '#111';
      ctx.font = '700 28px system-ui, sans-serif';
      ctx.fillText('Gaakstar Run', canvas.width/2 - 95, 110);
      ctx.font = '15px system-ui, sans-serif';
      ctx.fillStyle = isNight ? 'rgba(255,255,255,0.85)' : 'rgba(0,0,0,0.72)';
      ctx.fillText('Space / ↑ / 터치로 시작 & 점프', canvas.width/2 - 120, 140);
    }
  }

  function drawGameOver() {
    const isNight = Math.floor(score / NIGHT_EVERY) % 2 === 1;
    if (gameOver) {
      ctx.fillStyle = isNight ? 'rgba(0,0,0,0.55)' : 'rgba(255,255,255,0.75)';
      ctx.fillRect(0,0,canvas.width,canvas.height);
      ctx.fillStyle = isNight ? '#fff' : '#111';
      ctx.font = '700 34px system-ui, sans-serif';
      ctx.fillText('GAME OVER', canvas.width/2 - 105, 108);
      ctx.font = '15px system-ui, sans-serif';
      ctx.fillStyle = isNight ? 'rgba(255,255,255,0.9)' : 'rgba(0,0,0,0.75)';
      ctx.fillText('R 키 또는 화면 터치로 재시작', canvas.width/2 - 120, 140);
    }
  }

  function update(dt) {
    if (!running || gameOver) return;

    t += dt;
    speed = BASE_SPEED + t * SPEED_UP * BASE_SPEED * 60;

    // 점수는 속도에 비례해서 증가
    score += (speed * 0.12);
    if (score > best) {
      best = score;
      localStorage.setItem('gaakstar_best', String(Math.floor(best)));
    }

    // 장애물 생성
    if (nextObstX < canvas.width + 10) spawnObstacle();
    if (obstacles.length === 0) {
      spawnObstacle();
    }

    // 장애물 이동
    for (const o of obstacles) o.x -= speed;

    // 오래된 장애물 제거
    while (obstacles.length && obstacles[0].x + obstacles[0].w < -20) obstacles.shift();

    // 플레이어 물리
    player.vy += GRAVITY;
    player.y += player.vy;

    if (player.y >= GROUND_Y) {
      player.y = GROUND_Y;
      player.vy = 0;
      player.onGround = true;
    } else {
      player.onGround = false;
    }

    // 충돌
    const hb = getPlayerHitbox();
    for (const o of obstacles) {
      if (aabb(hb.x, hb.y, hb.w, hb.h, o.x, o.y, o.w, o.h)) {
        gameOver = true;
        running = false;
        break;
      }
    }

    // 다음 장애물 x 갱신
    if (obstacles.length) {
      const last = obstacles[obstacles.length - 1];
      nextObstX = Math.max(nextObstX, last.x + rand(OBST_MIN, OBST_MAX));
    }
  }

  function render() {
    drawBackground();

    // 장애물
    for (const o of obstacles) drawObstacle(o);

    // 플레이어
    drawPlayer();

    drawStartOverlay();
    drawGameOver();
  }

  let last = performance.now();
  function loop(now){
    const dt = clamp((now - last) / 16.67, 0, 2.5);
    last = now;
    update(dt);
    render();
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);

  function jump() {
    if (!running && !gameOver) start();
    if (gameOver) { reset(); return; }
    if (player.onGround) {
      player.vy = JUMP_V;
      player.onGround = false;
    }
  }

  function setDuck(on) {
    player.duck = on;
  }

  // ====== INPUT ======
  window.addEventListener('keydown', (e) => {
    if (e.code === 'Space' || e.code === 'ArrowUp') {
      e.preventDefault();
      jump();
    } else if (e.code === 'ArrowDown') {
      setDuck(true);
    } else if (e.key.toLowerCase() === 'r') {
      reset();
    }
  });
  window.addEventListener('keyup', (e) => {
    if (e.code === 'ArrowDown') setDuck(false);
  });

  // Mobile / touch
  canvas.addEventListener('pointerdown', (e) => {
    e.preventDefault();
    jump();
  });

  // init
  reset();
})();
</script>
</body>
</html>
