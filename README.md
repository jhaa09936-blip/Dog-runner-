# Dog-runner-
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Subbay Suffer - Dog Runner</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link
    rel="stylesheet"
    href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.3/css/all.min.css"
  />
  <style>
    #gameCanvas {
      touch-action: none;
    }
  </style>
</head>
<body class="bg-gradient-to-b from-blue-400 to-indigo-700 min-h-screen flex flex-col items-center justify-start p-4">
  <header class="w-full max-w-4xl flex items-center justify-between mb-6">
    <h1 class="text-white text-3xl font-extrabold tracking-wide">🐕 Subbay Suffer - Dog Runner</h1>
    <button id="startBtn" class="bg-yellow-400 hover:bg-yellow-500 text-indigo-900 font-bold py-2 px-4 rounded shadow-md flex items-center space-x-2">
      <i class="fas fa-play"></i>
      <span>Start Game</span>
    </button>
  </header>

  <main class="w-full max-w-4xl bg-indigo-900 rounded-lg shadow-lg p-4 flex flex-col items-center">
    <canvas
      id="gameCanvas"
      width="800"
      height="400"
      class="rounded-lg border-4 border-yellow-400 bg-gradient-to-b from-indigo-800 to-indigo-900"
    ></canvas>

    <div class="w-full mt-4 flex justify-between text-yellow-400 font-mono text-lg select-none">
      <div>Score: <span id="score">0</span></div>
      <div>High Score: <span id="highScore">0</span></div>
    </div>

    <div id="gameOverScreen" class="hidden mt-6 bg-yellow-400 text-indigo-900 rounded-lg p-6 w-full text-center font-bold text-2xl shadow-lg">
      Game Over! Your score: <span id="finalScore">0</span>
      <button id="restartBtn" class="ml-4 bg-indigo-900 text-yellow-400 px-4 py-2 rounded hover:bg-indigo-800 transition">Restart</button>
    </div>
  </main>

  <footer class="w-full max-w-4xl mt-10 text-yellow-300 text-center font-light select-none">
    <p>Press <b>Space</b> or Tap/Click to make the dog jump 🐕</p>
  </footer>

  <script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    const startBtn = document.getElementById("startBtn");
    const restartBtn = document.getElementById("restartBtn");
    const gameOverScreen = document.getElementById("gameOverScreen");
    const scoreEl = document.getElementById("score");
    const highScoreEl = document.getElementById("highScore");
    const finalScoreEl = document.getElementById("finalScore");

    const gravity = 0.6;
    const jumpForce = -12;
    const groundHeight = 80;

    let animationId;
    let gameRunning = false;
    let score = 0;
    let highScore = 0;

    // --- Dog character (only one) ---
    const dog = {
      x: canvas.width / 2 - 20,
      y: canvas.height - groundHeight - 40,
      width: 40,
      height: 40,
      velocityY: 0,
      jumping: false,
      color: "#FFD700", // golden
      draw() {
        ctx.fillStyle = this.color;
        ctx.fillRect(this.x, this.y, this.width, this.height);

        // Ears
        ctx.fillStyle = "#8B4513";
        ctx.fillRect(this.x + 5, this.y - 8, 8, 8);
        ctx.fillRect(this.x + 27, this.y - 8, 8, 8);

        // Eyes
        ctx.fillStyle = "#000";
        ctx.beginPath();
        ctx.arc(this.x + 12, this.y + 15, 4, 0, Math.PI * 2);
        ctx.arc(this.x + 28, this.y + 15, 4, 0, Math.PI * 2);
        ctx.fill();
      },
      update() {
        this.velocityY += gravity;
        this.y += this.velocityY;
        if (this.y + this.height > canvas.height - groundHeight) {
          this.y = canvas.height - groundHeight - this.height;
          this.velocityY = 0;
          this.jumping = false;
        }
      },
      jump() {
        if (!this.jumping) {
          this.velocityY = jumpForce;
          this.jumping = true;
        }
      },
    };

    // Obstacles
    class Obstacle {
      constructor() {
        this.width = 30 + Math.random() * 20;
        this.height = 30 + Math.random() * 50;
        this.x = canvas.width + this.width;
        this.y = canvas.height - groundHeight - this.height;
        this.color = "#EF4444"; // red
        this.speed = 6;
      }
      draw() {
        ctx.fillStyle = this.color;
        ctx.fillRect(this.x, this.y, this.width, this.height);
      }
      update() {
        this.x -= this.speed;
      }
    }

    let obstacles = [];
    let obstacleTimer = 0;
    let obstacleInterval = 90;

    function resetGame() {
      score = 0;
      obstacles = [];
      dog.y = canvas.height - groundHeight - dog.height;
      dog.velocityY = 0;
      dog.jumping = false;
      obstacleTimer = 0;
      gameOverScreen.classList.add("hidden");
      scoreEl.textContent = score;
    }

    function detectCollision(rect1, rect2) {
      return (
        rect1.x < rect2.x + rect2.width &&
        rect1.x + rect1.width > rect2.x &&
        rect1.y < rect2.y + rect2.height &&
        rect1.y + rect1.height > rect2.y
      );
    }

    function drawGround() {
      ctx.fillStyle = "#4F46E5";
      ctx.fillRect(0, canvas.height - groundHeight, canvas.width, groundHeight);
    }

    function gameLoop() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      drawGround();

      dog.update();
      dog.draw();

      obstacleTimer++;
      if (obstacleTimer > obstacleInterval) {
        obstacles.push(new Obstacle());
        obstacleTimer = 0;
        obstacleInterval = 60 + Math.floor(Math.random() * 60);
      }

      obstacles.forEach((obstacle, index) => {
        obstacle.update();
        obstacle.draw();

        if (obstacle.x + obstacle.width < 0) {
          obstacles.splice(index, 1);
          score++;
          scoreEl.textContent = score;
          if (score > highScore) {
            highScore = score;
            highScoreEl.textContent = highScore;
          }
        }

        if (detectCollision(dog, obstacle)) {
          endGame();
        }
      });

      if (gameRunning) animationId = requestAnimationFrame(gameLoop);
    }

    function endGame() {
      gameRunning = false;
      cancelAnimationFrame(animationId);
      finalScoreEl.textContent = score;
      gameOverScreen.classList.remove("hidden");
    }

    function startGame() {
      resetGame();
      gameRunning = true;
      gameLoop();
    }

    startBtn.addEventListener("click", () => {
      if (!gameRunning) startGame();
    });

    restartBtn.addEventListener("click", () => {
      if (!gameRunning) startGame();
    });

    // Controls (dog jumps)
    window.addEventListener("keydown", (e) => {
      if (e.code === "Space") {
        e.preventDefault();
        if (gameRunning) dog.jump();
      }
    });

    canvas.addEventListener("touchstart", (e) => {
      e.preventDefault();
      if (gameRunning) dog.jump();
    });

    canvas.addEventListener("mousedown", (e) => {
      e.preventDefault();
      if (gameRunning) dog.jump();
    });
  </script>
</body>
</html>
