<script lang="ts">
  import { onMount } from 'svelte';

  // Color palette from requirements
  const COLOR_PRIMARY = '#4EC0CA';   // sky
  const COLOR_SECONDARY = '#FFD700'; // pipes
  const COLOR_ACCENT = '#FF6F61';    // bird
  const COLOR_GROUND = '#C2B280';
  const PIPE_WIDTH = 52;
  const BIRD_SIZE = 36;
  const GRAVITY = 0.55;
  const JUMP = -8.5;
  const PIPE_GAP = 114;
  const PIPE_INTERVAL = 92;
  const BIRD_X = 56;
  const GAME_SPEED = 2.2;
  const GROUND_HEIGHT = 54;
  const MAX_PIPE_HEIGHT = 260;
  const MIN_PIPE_HEIGHT = 40;

  // Responsive base size
  let baseWidth = 400;
  let baseHeight = 600;

  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D;
  let animationFrame: number;
  let gameStarted = false;
  let gamePaused = false;
  let gameOver = false;
  let score = 0;
  let bestScore = 0;
  type Pipe = {
    x: number;
    width: number;
    height: number;
    gap: number;
    passed?: boolean;
  };
  let pipes: Pipe[] = [];
  let bird = {
    x: BIRD_X,
    y: baseHeight / 2,
    vy: 0,
    width: BIRD_SIZE,
    height: BIRD_SIZE
  };

  // Helper for touch
  let isTouchDevice: boolean = false;

  // PUBLIC_INTERFACE
  /** Start/restart the game */
  function startGame() {
    resetGame();
    gameStarted = true;
    gameOver = false;
    gamePaused = false;
    animate();
  }

  // PUBLIC_INTERFACE
  /** Pause or resume the game (not available if game is over) */
  function togglePause() {
    if (!gameStarted || gameOver) return;
    if (gamePaused) {
      gamePaused = false;
      animate();
    } else {
      gamePaused = true;
      cancelAnimationFrame(animationFrame);
    }
  }

  // PUBLIC_INTERFACE
  /**
   * Bird jump/flap handler
   */
  function flap() {
    if (!gameStarted) return;
    if (gameOver) return;
    bird.vy = JUMP;
  }

  function resetGame() {
    pipes = [];
    score = 0;
    bird = {
      x: BIRD_X,
      y: baseHeight / 2,
      vy: 0,
      width: BIRD_SIZE,
      height: BIRD_SIZE
    };
    addPipe();
  }

  function addPipe() {
    // top pipe height random within limits
    const topH = Math.floor(Math.random() * (MAX_PIPE_HEIGHT - MIN_PIPE_HEIGHT)) + MIN_PIPE_HEIGHT;
    pipes.push({
      x: baseWidth,
      width: PIPE_WIDTH,
      height: topH,
      gap: PIPE_GAP
    });
  }

  function animate() {
    if (gamePaused) return;

    update();
    draw();

    if (!gameOver) {
      animationFrame = requestAnimationFrame(animate);
    }
  }

  function update() {
    // bird physics
    bird.vy += GRAVITY;
    bird.y += bird.vy;

    // ground collision
    if (bird.y + BIRD_SIZE > baseHeight - GROUND_HEIGHT) {
      bird.y = baseHeight - GROUND_HEIGHT - BIRD_SIZE;
      endGame();
      return;
    }
    // ceiling
    if (bird.y < 0) bird.y = 0;

    // move pipes and respawn as needed
    for (let pipe of pipes) {
      pipe.x -= GAME_SPEED;
    }
    // Remove off-screen pipes
    if (pipes.length && pipes[0].x + PIPE_WIDTH < 0) {
      pipes.shift();
    }
    // Add pipes at interval
    if (pipes.length === 0 || pipes[pipes.length - 1].x < baseWidth - PIPE_INTERVAL) {
      addPipe();
    }

    // Collision & scoring
    for (let pipe of pipes) {
      // Each pipe is a pillar (upper and lower pipe)
      // Calculate the geometry of both
      const hitUpper = isColliding(
        bird.x, bird.y, bird.width, bird.height,
        pipe.x, 0, PIPE_WIDTH, pipe.height
      );
      const hitLower = isColliding(
        bird.x, bird.y, bird.width, bird.height,
        pipe.x, pipe.height + PIPE_GAP, PIPE_WIDTH, baseHeight - GROUND_HEIGHT - (pipe.height + PIPE_GAP)
      );
      if (hitUpper || hitLower) {
        endGame();
        return;
      }
      // Passing a pipe for score
      if (!pipe.passed && bird.x > pipe.x + PIPE_WIDTH) {
        score += 1;
        pipe.passed = true;
        // Update best score
        if (score > bestScore) {
          bestScore = score;
          localStorage.setItem('flappybird_best', bestScore + '');
        }
      }
    }
  }

  // Hitbox collision helper
  function isColliding(x1: number, y1: number, w1: number, h1: number,
    x2: number, y2: number, w2: number, h2: number) {
    return (
      x1 < x2 + w2 &&
      x1 + w1 > x2 &&
      y1 < y2 + h2 &&
      y1 + h1 > y2
    );
  }

  function endGame() {
    gameOver = true;
    cancelAnimationFrame(animationFrame);
  }

  function draw() {
    if (!ctx) return;
    ctx.save();
    ctx.clearRect(0, 0, baseWidth, baseHeight);

    // Draw Sky background
    ctx.fillStyle = COLOR_PRIMARY;
    ctx.fillRect(0, 0, baseWidth, baseHeight);

    // Pipes
    for (let pipe of pipes) {
      // Upper
      ctx.fillStyle = COLOR_SECONDARY;
      roundedRect(ctx, pipe.x, 0, PIPE_WIDTH, pipe.height, 10);
      ctx.fill();
      // Lower
      ctx.fillStyle = COLOR_SECONDARY;
      roundedRect(ctx, pipe.x, pipe.height + PIPE_GAP, PIPE_WIDTH, baseHeight - GROUND_HEIGHT - (pipe.height + PIPE_GAP), 10);
      ctx.fill();

      // pipe edge accent
      ctx.strokeStyle = '#B29600';
      ctx.lineWidth = 2;
      ctx.strokeRect(pipe.x, 0, PIPE_WIDTH, pipe.height);
      ctx.strokeRect(pipe.x, pipe.height + PIPE_GAP, PIPE_WIDTH, baseHeight - GROUND_HEIGHT - (pipe.height + PIPE_GAP));
    }

    // Draw ground
    ctx.fillStyle = COLOR_GROUND;
    ctx.fillRect(0, baseHeight - GROUND_HEIGHT, baseWidth, GROUND_HEIGHT);

    // Draw bird -- simple cartoony circle and beak
    ctx.save();
    ctx.translate(bird.x + BIRD_SIZE / 2, bird.y + BIRD_SIZE / 2);
    ctx.rotate(bird.vy * 0.03); // Tilt for flapping effect

    // Bird body
    ctx.beginPath();
    ctx.arc(0, 0, BIRD_SIZE / 2, 0, Math.PI * 2, false);
    ctx.fillStyle = COLOR_ACCENT;
    ctx.fill();

    // Eye
    ctx.beginPath();
    ctx.arc(8, -7, 6, 0, Math.PI * 2);
    ctx.fillStyle = '#fff';
    ctx.fill();
    ctx.beginPath();
    ctx.arc(10, -7, 2.5, 0, Math.PI * 2);
    ctx.fillStyle = '#222';
    ctx.fill();

    // Wing
    ctx.beginPath();
    ctx.ellipse(-6, 0, 7, 12, Math.PI / 6, 0, Math.PI * 2);
    ctx.fillStyle = '#FFD8B4';
    ctx.fill();

    // Beak
    ctx.beginPath();
    ctx.moveTo(BIRD_SIZE / 2 - 2, 2);
    ctx.lineTo(BIRD_SIZE / 2 + 12, -1);
    ctx.lineTo(BIRD_SIZE / 2 - 2, -6);
    ctx.closePath();
    ctx.fillStyle = '#FFA500';
    ctx.fill();

    ctx.restore();

    // Score
    ctx.font = "32px Arial";
    ctx.fillStyle = '#fff';
    ctx.strokeStyle = '#333';
    ctx.lineWidth = 3;
    ctx.textAlign = "center";
    ctx.strokeText(score + '', baseWidth / 2, 70);
    ctx.fillText(score + '', baseWidth / 2, 70);

    // Best score display (small top right)
    ctx.font = "16px Arial";
    ctx.fillStyle = '#fff';
    ctx.textAlign = "right";
    ctx.fillText("Best: " + bestScore, baseWidth - 16, 32);

    // Overlay UI for start/paused/game over
    if (!gameStarted) {
      ctx.save();
      ctx.globalAlpha = 0.95;
      ctx.fillStyle = "#ffffffcc";
      ctx.fillRect(0, 0, baseWidth, baseHeight);
      ctx.globalAlpha = 1;
      ctx.fillStyle = COLOR_ACCENT;
      ctx.font = "bold 34px Arial";
      ctx.textAlign = "center";
      ctx.fillText("Flappy Bird", baseWidth / 2, baseHeight / 2 - 52);
      ctx.fillStyle = COLOR_PRIMARY;
      ctx.font = "18px Arial";
      ctx.fillText("Tap or Space to Start", baseWidth / 2, baseHeight / 2 - 18);
      ctx.restore();
    } else if (gamePaused && !gameOver) {
      ctx.save();
      ctx.globalAlpha = 0.7;
      ctx.fillStyle = "#fff";
      ctx.fillRect(0, 0, baseWidth, baseHeight);
      ctx.globalAlpha = 1;
      ctx.fillStyle = COLOR_ACCENT;
      ctx.font = "bold 30px Arial";
      ctx.textAlign = "center";
      ctx.fillText("Paused", baseWidth / 2, baseHeight / 2 - 25);
      ctx.fillStyle = COLOR_PRIMARY;
      ctx.font = "19px Arial";
      ctx.fillText("Press Space or Tap to Resume", baseWidth / 2, baseHeight / 2 + 12);
      ctx.restore();
    } else if (gameOver) {
      ctx.save();
      ctx.globalAlpha = 0.92;
      ctx.fillStyle = "#ffffffc0";
      ctx.fillRect(0, 0, baseWidth, baseHeight);
      ctx.globalAlpha = 1;
      ctx.fillStyle = COLOR_ACCENT;
      ctx.font = "bold 34px Arial";
      ctx.textAlign = "center";
      ctx.fillText("Game Over", baseWidth / 2, baseHeight / 2 - 36);
      ctx.fillStyle = COLOR_PRIMARY;
      ctx.font = "18px Arial";
      ctx.fillText("Score: " + score, baseWidth / 2, baseHeight / 2);
      ctx.font = "18px Arial";
      ctx.fillStyle = COLOR_ACCENT;
      ctx.fillText("Tap or Space to Restart", baseWidth / 2, baseHeight / 2 + 30);
      ctx.restore();
    }

    ctx.restore();
  }

  function roundedRect(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.lineTo(x + w - r, y);
    ctx.quadraticCurveTo(x + w, y, x + w, y + r);
    ctx.lineTo(x + w, y + h - r);
    ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
    ctx.lineTo(x + r, y + h);
    ctx.quadraticCurveTo(x, y + h, x, y + h - r);
    ctx.lineTo(x, y + r);
    ctx.quadraticCurveTo(x, y, x + r, y);
    ctx.closePath();
  }

  // Resize game for responsiveness
  function resizeCanvas() {
    let width = window.innerWidth > 480 ? 400 : Math.min(400, window.innerWidth - 20);
    let height = (width / 400) * 600;
    baseWidth = width;
    baseHeight = height;
    if (canvas) {
      canvas.width = baseWidth;
      canvas.height = baseHeight;
    }
    if (gameStarted || gameOver) draw();
  }

  function handleStartInput() {
    if (!gameStarted || (gameOver && !gamePaused)) {
      startGame();
    } else if (gamePaused) {
      togglePause();
    } else {
      flap();
    }
  }

  // Keyboard/touch event listeners
  function onKeydown(e: KeyboardEvent) {
    if ([" ", "ArrowUp"].includes(e.key)) {
      e.preventDefault();
      if (!gameStarted || gameOver) {
        startGame();
      } else if (gamePaused) {
        togglePause();
      } else {
        flap();
      }
    }
    if (e.key.toLowerCase() === 'p' && gameStarted && !gameOver) {
      togglePause();
    }
    if (e.key.toLowerCase() === 'r' && gameOver) {
      startGame();
    }
  }

  function onTouchStart(e: TouchEvent) {
    e.preventDefault();
    handleStartInput();
  }

  // Handle clicking on the canvas (desktop browser mode)
  function onClick() {
    handleStartInput();
  }

  onMount(() => {
    ctx = canvas.getContext('2d');
    // Responsive initial sizing
    resizeCanvas();
    window.addEventListener('resize', resizeCanvas);

    // Detect touch device
    isTouchDevice = 'ontouchstart' in window;

    // Controls
    window.addEventListener('keydown', onKeydown);
    canvas.addEventListener('touchstart', onTouchStart, { passive: false });
    canvas.addEventListener('mousedown', onClick);

    // Best score from localStorage
    let raw = localStorage.getItem('flappybird_best');
    bestScore = raw ? parseInt(raw) : 0;

    // Draw static intro UI
    draw();

    return () => {
      window.removeEventListener('resize', resizeCanvas);
      window.removeEventListener('keydown', onKeydown);
      canvas.removeEventListener('touchstart', onTouchStart);
      canvas.removeEventListener('mousedown', onClick);
      cancelAnimationFrame(animationFrame);
    };
  });

</script>

<div class="flappy-wrapper">
  <div class="flappy-title">
    <h1>Flappy Bird</h1>
    <div class="scores">
      <span>Score: {score}</span>
      <span>Best: {bestScore}</span>
    </div>
  </div>
  <div class="canvas-container">
    <canvas
      bind:this={canvas}
      tabindex="0"
      width={baseWidth}
      height={baseHeight}
      aria-label="Flappy Bird Game"
      class:touch-device={isTouchDevice}
      style="background: #eee; border-radius: 12px; box-shadow: 0 2px 18px #0003;"
      />
    <div class="overlay-buttons">
      {#if !gameStarted || gameOver}
        <button class="main"
          on:click={startGame}
          aria-label={gameOver ? "Restart Game" : "Start Game"}>
          {gameOver ? 'Restart' : 'Start'}
        </button>
      {:else}
        <button class="main" on:click={togglePause} aria-label="Pause/Resume Game">
          {gamePaused ? 'Resume' : 'Pause'}
        </button>
      {/if}
    </div>
    <div class="hint">
      <span>
        <b>Controls:</b>
        <span class="for-desktop">Space/Up = Flap | P = Pause | R = Restart</span>
        <span class="for-touch">Tap = Flap or (Re)Start</span>
      </span>
    </div>
  </div>
</div>

<style>
.flappy-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100vw;
  max-width: 480px;
  margin: 0 auto;
  user-select: none;
}
.flappy-title {
  text-align: center;
  margin: 0.5em 0 0.7em 0;
}
.flappy-title h1 {
  font-size: 2rem;
  color: #222;
  margin-bottom: 0.28em;
  font-family: 'Arial Rounded MT', Arial, sans-serif;
  text-shadow: 1px 3px 2px #ffffffb0;
}
.scores {
  font-size: 1.1em;
  color: #234;
  display: flex;
  gap: 2em;
  justify-content: center;
}
.canvas-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: #fff;
  box-shadow: 0 3px 24px #0004;
  border-radius: 12px;
  padding: 12px 10px 7px 10px;
  margin: 0 0 .9em 0;
  width: 100%;
  max-width: 430px;
}

canvas {
  width: 100%;
  height: auto;
  aspect-ratio: 2/3;
  background: #eee;
}

.overlay-buttons {
  margin: .6em auto 0.44em auto;
  display: flex;
  justify-content: center;
  width: 100%;
}
button.main {
  background: linear-gradient(0deg, var(--primary, #4EC0CA) 65%, #64d6ec 100%);
  color: #fff;
  font-weight: bold;
  border: none;
  border-radius: 30px;
  box-shadow: 0 2px 7px #0002;
  padding: .6em 2.4em;
  font-size: 1.3rem;
  cursor: pointer;
  transition: background .18s, color .18s;
  font-family: inherit;
  margin: 0 6px;
}
button.main:hover, button.main:focus {
  background: var(--accent, #FF6F61);
  color: #fffbe8;
  outline: none;
}
.hint {
  font-size: 0.98em;
  color: #446b8c;
  opacity: .90;
  margin: 0 0 7px 0;
  text-align: center;
}
.for-touch { display: none; }
@media (hover: none), (pointer: coarse) {
  .for-desktop { display: none; }
  .for-touch { display: inline; }
}
@media (max-width: 480px) {
  .flappy-title h1 { font-size: 1.45rem; }
  .canvas-container { padding: 5px 1px 2px 1px; }
}
</style>
