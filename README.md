<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>🚀 Космическая змейка</title>
  <style>
    body {
      margin: 0;
      padding: 10px;
      text-align: center;
      font-family: Arial, sans-serif;
      background: #000;
      color: #0f0;
      touch-action: manipulation;
    }
    h1 {
      color: #0f0;
      font-size: 24px;
      margin: 10px 0;
    }
    #score, #best {
      font-size: 20px;
      font-weight: bold;
      margin: 5px 0;
    }
    #best {
      color: gold;
    }
    canvas {
      background: #000;
      border: 1px solid #333;
      image-rendering: pixelated;
      display: block;
      margin: 20px auto;
      width: 300px;
      height: 300px;
    }
    .controls {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      grid-template-rows: repeat(3, 1fr);
      gap: 10px;
      width: 200px;
      margin: 20px auto;
    }
    .btn {
      background: #0066cc;
      color: white;
      border: none;
      padding: 24px;
      font-size: 24px;
      border-radius: 10px;
      cursor: pointer;
    }
    button#restart-btn {
      padding: 12px 24px;
      font-size: 18px;
      background: #00cc44;
      color: white;
      border: none;
      border-radius: 6px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <h1>🚀 КОСМИЧЕСКАЯ ЗМЕЙКА</h1>
  <div id="score">Очки: 0</div>
  <div id="best">Рекорд: 0</div>
  <canvas id="game" width="300" height="300"></canvas>
  <div class="controls">
    <button class="btn"></button>
    <button class="btn" onclick="setDir(0,-1)">↑</button>
    <button class="btn"></button>
    <button class="btn" onclick="setDir(-1,0)">←</button>
    <button class="btn">●</button>
    <button class="btn" onclick="setDir(1,0)">→</button>
    <button class="btn"></button>
    <button class="btn" onclick="setDir(0,1)">↓</button>
    <button class="btn"></button>
  </div>
  <button id="restart-btn" onclick="restart()">Начать заново</button>

  <script>
    // Безопасное подключение к Telegram
    const tg = window.Telegram?.WebApp;
    if (tg) {
      try {
        tg.ready();
        tg.expand();
      } catch (e) {
        console.warn("Telegram WebApp недоступен", e);
      }
    }

    // Элементы
    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");
    const scoreDisplay = document.getElementById("score");
    const bestDisplay = document.getElementById("best");

    // Настройки
    const gridSize = 15;
    const tileCount = 20;
    const moveSpeed = 5;

    // Звёзды
    const stars = [];
    for (let i = 0; i < 20; i++) {
      stars.push({
        x: Math.random() * canvas.width,
        y: Math.random() * canvas.height,
        r: Math.random() * 1.5 + 0.5,
        a: Math.random() * 0.8 + 0.2
      });
    }

    // Фрукты
    const FRUITS = [
      { emoji: "🍎", points: 5, weight: 10 },
      { emoji: "🍌", points: 15, weight: 8 },
      { emoji: "🍒", points: 10, weight: 9 },
      { emoji: "🍓", points: 12, weight: 7 },
      { emoji: "🥝", points: 10, weight: 8 },
      { emoji: "🍇", points: 14, weight: 6 },
      { emoji: "🍊", points: 11, weight: 7 },
      { emoji: "🍉", points: 25, weight: 3 },
      { emoji: "🍈", points: 22, weight: 3 },
      { emoji: "🥑", points: 35, weight: 2 },
      { emoji: "🥕", points: 50, weight: 1 }
    ];

    // Состояние
    let snake = [{ x: 10, y: 10 }];
    let snakePos = { x: 10 * gridSize, y: 10 * gridSize };
    let dx = 1, dy = 0;
    let nextDx = 1, nextDy = 0;
    let score = 0;
    let bestScore = 0;
    let food = null;
    let gameRunning = false;
    let gameStarted = false;
    let gameInterval = null;
    let popups = [];
    let eatEffects = [];
    let celebration = null;

    // Загрузка рекорда
    try {
      bestScore = parseInt(localStorage.getItem("spaceSnakeBest")) || 0;
    } catch (e) {
      console.warn("localStorage недоступен", e);
    }
    if (bestDisplay) bestDisplay.textContent = `Рекорд: ${bestScore}`;

    // Всплывающие очки
    function addPopup(x, y, text, color = "#00ff00") {
      popups.push({
        x: x * gridSize + gridSize / 2,
        y: y * gridSize,
        text: text,
        color: color,
        alpha: 1,
        dy: -1.5
      });
    }

    // Эффект поедания
    function addEatEffect(x, y) {
      for (let i = 0; i < 5; i++) {
        eatEffects.push({
          x: x * gridSize + gridSize / 2,
          y: y * gridSize + gridSize / 2,
          dx: (Math.random() - 0.5) * 8,
          dy: (Math.random() - 0.5) * 8,
          alpha: 1,
          r: Math.random() * 2 + 1
        });
      }
    }

    // Анимация уровня
    function startCelebration() {
      celebration = { text: "🚀 LEVEL UP!", alpha: 1 };
      setTimeout(() => { celebration = null; }, 1500);
    }

    // Выбор фрукта
    function getRandomFruit() {
      const totalWeight = FRUITS.reduce((sum, f) => sum + f.weight, 0);
      let rand = Math.random() * totalWeight;
      for (let fruit of FRUITS) {
        rand -= fruit.weight;
        if (rand <= 0) return { ...fruit };
      }
      return FRUITS[0];
    }

    // Размещение еды
    function placeFood() {
      food = getRandomFruit();
      food.gridX = Math.floor(Math.random() * tileCount);
      food.gridY = Math.floor(Math.random() * tileCount);

      for (let part of snake) {
        if (part.x === food.gridX && part.y === food.gridY) {
          placeFood();
          return;
        }
      }
    }

    // Фон с звёздами
    function drawSpace() {
      ctx.fillStyle = "black";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      stars.forEach(s => {
        ctx.fillStyle = `rgba(255, 255, 255, ${s.a})`;
        ctx.beginPath();
        ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
        ctx.fill();
      });
    }

    // Основная отрисовка
    function draw() {
      drawSpace();

      // Змейка
      for (let i = 0; i < snake.length; i++) {
        const part = snake[i];
        const x = part.x * gridSize;
        const y = part.y * gridSize;

        ctx.fillStyle = i === 0 ? "#00ff44" : "#00cc22";
        ctx.beginPath();
        ctx.arc(x + gridSize/2, y + gridSize/2, gridSize/2 - 1, 0, Math.PI * 2);
        ctx.fill();

        // Глаза
        if (i === 0) {
          ctx.fillStyle = "white";
          const eyeSize = 2;
          const eyeX = dx === 1 ? x + gridSize - 5 : dx === -1 ? x + 5 : x + 7;
          const eyeY = dy === 1 ? y + gridSize - 5 : dy === -1 ? y + 5 : y + 7;
          ctx.beginPath();
          ctx.arc(eyeX, eyeY, eyeSize, 0, Math.PI * 2);
          ctx.fill();
          ctx.fillStyle = "black";
          ctx.beginPath();
          ctx.arc(eyeX + (dx || 0), eyeY + (dy || 0), 0.8, 0, Math.PI * 2);
          ctx.fill();
        }
      }

      // Фрукт
      if (food) {
        ctx.font = `${gridSize}px Arial`;
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";
        ctx.fillText(
          food.emoji,
          food.gridX * gridSize + gridSize / 2,
          food.gridY * gridSize + gridSize / 2
        );
      }

      // Всплывающие очки
      popups.forEach((p, i) => {
        ctx.fillStyle = `rgba(${p.color === "yellow" ? "255,255,0" : "0,255,0"}, ${p.alpha})`;
        ctx.font = "bold 16px Arial";
        ctx.fillText(p.text, p.x, p.y);
        p.y += p.dy;
        p.alpha -= 0.02;
        if (p.alpha <= 0) popups.splice(i, 1);
      });

      // Эффекты
      eatEffects.forEach((e, i) => {
        ctx.fillStyle = `rgba(255, 255, 0, ${e.alpha})`;
        ctx.beginPath();
        ctx.arc(e.x, e.y, e.r, 0, Math.PI * 2);
        ctx.fill();
        e.x += e.dx;
        e.y += e.dy;
        e.alpha -= 0.03;
        if (e.alpha <= 0) eatEffects.splice(i, 1);
      });

      // Анимация уровня
      if (celebration) {
        ctx.fillStyle = `rgba(255, 215, 0, ${celebration.alpha})`;
        ctx.font = "bold 24px Arial";
        const w = ctx.measureText(celebration.text).width;
        ctx.fillText(celebration.text, (canvas.width - w) / 2, 150);
        if (celebration.alpha > 0) celebration.alpha -= 0.01;
      }
    }

    // Управление
    function setDir(x, y) {
      if (!gameStarted) {
        gameStarted = true;
        gameRunning = true;
        clearInterval(gameInterval);
        gameInterval = setInterval(gameLoop, 80);
      }

      // Блокировка разворота
      if (x === 1 && dx === -1) return;
      if (x === -1 && dx === 1) return;
      if (y === 1 && dy === -1) return;
      if (y === -1 && dy === 1) return;

      nextDx = x;
      nextDy = y;
    }

    // Управление с клавиатуры
    document.addEventListener("keydown", (e) => {
      if (e.key === "ArrowLeft") setDir(-1, 0);
      if (e.key === "ArrowRight") setDir(1, 0);
      if (e.key === "ArrowUp") setDir(0, -1);
      if (e.key === "ArrowDown") setDir(0, 1);
    });

    // Основной цикл
    function gameLoop() {
      if (!gameRunning) return;

      // Применяем направление
      if (nextDx !== 0 || nextDy !== 0) {
        dx = nextDx;
        dy = nextDy;
      }

      // Двигаем позицию
      snakePos.x += dx * moveSpeed;
      snakePos.y += dy * moveSpeed;

      // Проверка перехода на новую клетку
      if (
        Math.abs(snakePos.x - snake[0].x * gridSize) >= gridSize ||
        Math.abs(snakePos.y - snake[0].y * gridSize) >= gridSize
      ) {
        const head = { x: snake[0].x + dx, y: snake[0].y + dy };

        // Проход сквозь стену
        if (head.x < 0) head.x = tileCount - 1;
        if (head.x >= tileCount) head.x = 0;
        if (head.y < 0) head.y = tileCount - 1;
        if (head.y >= tileCount) head.y = 0;

        // Самопересечение
        for (let part of snake) {
          if (part.x === head.x && part.y === head.y) {
            gameOver();
            return;
          }
        }

        snake.unshift(head);
        snakePos.x = head.x * gridSize;
        snakePos.y = head.y * gridSize;

        // Проверка еды
        if (head.x === food.gridX && head.y === food.gridY) {
          const points = food.points;
          score += points;
          if (scoreDisplay) scoreDisplay.textContent = `Очки: ${score}`;
          addPopup(head.x, head.y, `+${points}`, points > 20 ? "yellow" : "green");
          addEatEffect(head.x, head.y);
          placeFood();
          if (score % 100 === 0 && score <= 1000) startCelebration();
        } else {
          snake.pop();
        }
      }

      draw();
    }

    // Конец игры
    function gameOver() {
      gameRunning = false;
      clearInterval(gameInterval);
      try {
        if (score > bestScore) {
          bestScore = score;
          localStorage.setItem("spaceSnakeBest", bestScore);
          if (bestDisplay) bestDisplay.textContent = `Рекорд: ${bestScore} 🌟`;
        }
      } catch (e) {}

      drawSpace();
      ctx.fillStyle = "rgba(0,0,0,0.8)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "red";
      ctx.font = "30px Arial";
      const t1 = "МИССИЯ";
      const t2 = "ЗАВЕРШЕНА";
      ctx.fillText(t1, (canvas.width - ctx.measureText(t1).width) / 2, 130);
      ctx.fillText(t2, (canvas.width - ctx.measureText(t2).width) / 2, 170);
      ctx.fillStyle = "white";
      ctx.font = "20px Arial";
      ctx.fillText(`Очки: ${score}`, (canvas.width - ctx.measureText(`Очки: ${score}`).width) / 2, 210);
    }

    // Перезапуск
    function restart() {
      clearInterval(gameInterval);
      snake = [{ x: 10, y: 10 }];
      snakePos = { x: 10 * gridSize, y: 10 * gridSize };
      dx = 1; dy = 0;
      nextDx = 1; nextDy = 0;
      score = 0;
      if (scoreDisplay) scoreDisplay.textContent = "Очки: 0";
      gameRunning = false;
      gameStarted = false;
      popups = [];
      eatEffects = [];
      celebration = null;
      placeFood();
      draw();
      showStartAnimation();
    }

    // Анимация старта
    function showStartAnimation() {
      let step = 0;
      const texts = ["включаем двигатели...", "выход в эфир...", "змейка-корабль, старт!"];
      const anim = setInterval(() => {
        drawSpace();
        ctx.fillStyle = "rgba(0,0,0,0.7)";
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        ctx.fillStyle = "#00ff00";
        ctx.font = "20px Arial";
        const title = "🚀 КОСМОС";
        const tw = ctx.measureText(title).width;
        ctx.fillText(title, (canvas.width - tw) / 2, 120);

        ctx.fillStyle = "white";
        ctx.font = "16px Arial";
        const text = texts[step % 3];
        const w = ctx.measureText(text).width;
        ctx.fillText(text, (canvas.width - w) / 2, 150);

        step++;
        if (step > 12) {
          clearInterval(anim);
          draw();
          showHint();
        }
      }, 250);
    }

    function showHint() {
      ctx.fillStyle = "rgba(0,0,0,0.5)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "white";
      ctx.font = "18px Arial";
      const t1 = "Нажми стрелку";
      const t2 = "для старта";
      ctx.fillText(t1, (canvas.width - ctx.measureText(t1).width) / 2, 140);
      ctx.fillText(t2, (canvas.width - ctx.measureText(t2).width) / 2, 170);
    }

    // Старт
    window.addEventListener('load', () => {
      try {
        placeFood();
        draw();
        showStartAnimation();
      } catch (e) {
        console.error("Ошибка при загрузке:", e);
        alert("Ошибка: " + e.message);
      }
    });
  </script>
</body>
</html>
