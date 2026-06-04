<template>
  <div class="arcade-container" :class="selectedTheme" @click="initAudio">
    <!-- Header -->
    <header class="arcade-header">
      <h1 class="logo-text">NEON <span class="accent-text">SNAKE</span></h1>
      <p class="subtitle-text">VIRTUAL ARCADE CABINET</p>
    </header>

    <!-- Main Workspace -->
    <main class="arcade-main">
      <!-- LEFT SECTION: Play Area -->
      <section class="game-section">
        <!-- Stats HUD -->
        <div class="hud-container">
          <div class="hud-stat score-hud">
            <span class="hud-label">SCORE</span>
            <span class="hud-value">{{ score }}</span>
          </div>
          <div class="hud-stat highscore-hud">
            <span class="hud-label">HIGH SCORE</span>
            <span class="hud-value">{{ highScore }}</span>
          </div>
        </div>

        <!-- Canvas Board Wrapper -->
        <div class="board-wrapper">
          <canvas
            ref="canvasRef"
            width="600"
            height="600"
            class="game-canvas"
            @touchstart="handleTouchStart"
            @touchend="handleTouchEnd"
          ></canvas>

          <!-- Screens Overlay (Menu, Pause, Gameover) -->
          <!-- Menu Screen -->
          <div v-if="!gameStarted" class="screen-overlay menu-screen">
            <h2 class="screen-title pulse-glow">INSERT COIN</h2>
            <p class="screen-subtitle">Press SPACE or Click PLAY to Start</p>
            <button class="arcade-btn play-btn" @click="startGame">PLAY GAME</button>
            <p class="screen-controls-hint">Use WASD / Arrow Keys or Swipe to steer</p>
          </div>

          <!-- Pause Screen -->
          <div v-else-if="isPaused" class="screen-overlay pause-screen">
            <h2 class="screen-title">GAME PAUSED</h2>
            <p class="screen-subtitle">Press SPACE to Resume</p>
            <div class="overlay-buttons">
              <button class="arcade-btn" @click="togglePause">RESUME</button>
              <button class="arcade-btn danger" @click="quitToMenu">QUIT TO MENU</button>
            </div>
          </div>

          <!-- Game Over Screen -->
          <div v-else-if="gameOver" class="screen-overlay gameover-screen">
            <h2 class="screen-title game-over-text">GAME OVER</h2>
            <div class="run-stats">
              <p>Final Score: <span class="highlight">{{ score }}</span></p>
              <p v-if="isNewHighScore" class="new-record pulse-glow">NEW HIGH SCORE!</p>
            </div>

            <!-- Leaderboard Entry Form -->
            <div v-if="qualifiesForLeaderboard && !scoreSaved" class="leaderboard-entry">
              <p>You made the top 5! Enter your initials:</p>
              <div class="entry-row">
                <input
                  v-model="playerInitials"
                  type="text"
                  maxlength="3"
                  class="initials-input"
                  placeholder="AAA"
                  @keyup.enter="submitScore"
                />
                <button class="arcade-btn mini submit-btn" @click="submitScore">SUBMIT</button>
              </div>
            </div>

            <div class="overlay-buttons">
              <button class="arcade-btn play-btn" @click="startGame">PLAY AGAIN</button>
              <button class="arcade-btn" @click="quitToMenu">MAIN MENU</button>
            </div>
          </div>
        </div>

        <!-- Powerups Active Bar HUD -->
        <div class="powerups-hud" v-if="gameStarted && !gameOver && activePowerups.length > 0">
          <div v-for="p in activePowerups" :key="p.type" class="powerup-badge" :class="p.type">
            <span class="powerup-icon">
              <svg v-if="p.type === 'shield'" viewBox="0 0 24 24" width="16" height="16"><path fill="currentColor" d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z"/></svg>
              <svg v-else-if="p.type === 'speed'" viewBox="0 0 24 24" width="16" height="16"><path fill="currentColor" d="M13 2H6v10h4v10l9-12h-6V2z"/></svg>
              <svg v-else-if="p.type === 'slowmo'" viewBox="0 0 24 24" width="16" height="16"><path fill="currentColor" d="M12 2C6.5 2 2 6.5 2 12s4.5 10 10 10 10-4.5 10-10S17.5 2 12 2zm0 18c-4.4 0-8-3.6-8-8s3.6-8 8-8 8 3.6 8 8-3.6 8-8 8zm-1-13h2v6h-2zm0 8h2v2h-2z"/></svg>
            </span>
            <span class="powerup-name">{{ p.type.toUpperCase() }}</span>
            <div class="powerup-progress">
              <div class="powerup-bar" :style="{ width: p.type === 'shield' ? '100%' : (p.duration / p.maxDuration * 100) + '%' }"></div>
            </div>
          </div>
        </div>

        <!-- Mobile Touch Controllers / On Screen D-Pad -->
        <div class="mobile-controls-panel">
          <button class="toggle-dpad-btn" @click="showDPad = !showDPad">
            {{ showDPad ? 'Hide Controls' : 'Show On-Screen Controls' }}
          </button>
          
          <div v-if="showDPad" class="dpad-container">
            <div class="dpad-row">
              <button class="dpad-btn up" @click="changeDirection({x: 0, y: -1})">▲</button>
            </div>
            <div class="dpad-row">
              <button class="dpad-btn left" @click="changeDirection({x: -1, y: 0})">◀</button>
              <div class="dpad-center"></div>
              <button class="dpad-btn right" @click="changeDirection({x: 1, y: 0})">▶</button>
            </div>
            <div class="dpad-row">
              <button class="dpad-btn down" @click="changeDirection({x: 0, y: 1})">▼</button>
            </div>
          </div>
        </div>
      </section>

      <!-- RIGHT SECTION: Dashboard (Settings, Customization, Scores) -->
      <section class="dashboard-section">
        <!-- Dashboard Tabs -->
        <nav class="dashboard-tabs">
          <button
            class="tab-btn"
            :class="{ active: activeTab === 'settings' }"
            @click="activeTab = 'settings'"
          >
            SETTINGS
          </button>
          <button
            class="tab-btn"
            :class="{ active: activeTab === 'skins' }"
            @click="activeTab = 'skins'"
          >
            SKINS & THEMES
          </button>
          <button
            class="tab-btn"
            :class="{ active: activeTab === 'scores' }"
            @click="activeTab = 'scores'"
          >
            LEADERBOARD
          </button>
        </nav>

        <!-- Tab 1: Settings -->
        <div v-if="activeTab === 'settings'" class="tab-content settings-tab">
          <div class="setting-group">
            <h3>DIFFICULTY</h3>
            <div class="btn-group">
              <button
                v-for="d in ['easy', 'medium', 'hard']"
                :key="d"
                class="arcade-btn mini"
                :class="{ active: difficulty === d }"
                @click="setDifficulty(d)"
              >
                {{ d.toUpperCase() }}
              </button>
            </div>
          </div>

          <div class="setting-group">
            <h3>WALL COLLISIONS</h3>
            <div class="btn-group">
              <button
                class="arcade-btn mini"
                :class="{ active: wrapAround }"
                @click="setWrapAround(true)"
              >
                WRAP-AROUND
              </button>
              <button
                class="arcade-btn mini"
                :class="{ active: !wrapAround }"
                @click="setWrapAround(false)"
              >
                SOLID WALLS
              </button>
            </div>
            <p class="setting-desc">
              {{ wrapAround ? 'Snake travels through walls and re-enters from the opposite side.' : 'Colliding with walls ends the game (unless shielded).' }}
            </p>
          </div>

          <div class="setting-group">
            <h3>AUDIO SYSTEM</h3>
            <button
              class="arcade-btn mini mute-btn"
              :class="{ warning: soundMuted }"
              @click="toggleMute"
            >
              {{ soundMuted ? 'UNMUTE SOUNDS' : 'MUTE SOUNDS' }}
            </button>
          </div>
        </div>

        <!-- Tab 2: Customization (Skins & Themes) -->
        <div v-if="activeTab === 'skins'" class="tab-content skins-tab">
          <div class="setting-group">
            <h3>SNAKE SKINS</h3>
            <div class="skins-grid">
              <div
                v-for="skin in skinOptions"
                :key="skin.id"
                class="skin-card"
                :class="{ active: selectedSkin === skin.id }"
                @click="selectSkin(skin.id)"
              >
                <div class="skin-preview" :style="{ background: skin.color }"></div>
                <span class="skin-name">{{ skin.name }}</span>
              </div>
            </div>
          </div>

          <div class="setting-group">
            <h3>BOARD THEMES</h3>
            <div class="themes-grid">
              <div
                v-for="theme in themeOptions"
                :key="theme.id"
                class="theme-card"
                :class="{ active: selectedTheme === theme.id }"
                @click="selectTheme(theme.id)"
              >
                <div class="theme-colors-preview">
                  <div class="theme-color" :style="{ background: theme.colors.bg }"></div>
                  <div class="theme-color" :style="{ background: theme.colors.glow }"></div>
                </div>
                <span class="theme-name">{{ theme.name }}</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Tab 3: Leaderboard & Stats -->
        <div v-if="activeTab === 'scores'" class="tab-content scores-tab">
          <div class="setting-group">
            <h3>HIGH SCORE BOARD</h3>
            <div class="leaderboard-table">
              <div class="table-header">
                <span>RANK</span>
                <span>INITIALS</span>
                <span>SCORE</span>
              </div>
              <div
                v-for="(entry, idx) in leaderboard"
                :key="idx"
                class="table-row"
                :class="{ gold: idx === 0, silver: idx === 1, bronze: idx === 2 }"
              >
                <span class="rank-col">
                  <span v-if="idx === 0">🏆</span>
                  <span v-else-if="idx === 1">🥈</span>
                  <span v-else-if="idx === 2">🥉</span>
                  <span v-else>{{ idx + 1 }}</span>
                </span>
                <span class="initials-col">{{ entry.name }}</span>
                <span class="score-col">{{ entry.score }}</span>
              </div>
              <div v-if="leaderboard.length === 0" class="no-records">
                NO RECORDED SCORES YET
              </div>
            </div>
          </div>

          <div class="setting-group">
            <h3>ARCADE STATS</h3>
            <div class="stats-grid">
              <div class="stat-box">
                <span class="stat-value">{{ gameStats.gamesPlayed }}</span>
                <span class="stat-label">GAMES PLAYED</span>
              </div>
              <div class="stat-box">
                <span class="stat-value">{{ gameStats.totalFoodEaten }}</span>
                <span class="stat-label">APPLES EATEN</span>
              </div>
              <div class="stat-box">
                <span class="stat-value">{{ formatTime(gameStats.totalTimePlayed) }}</span>
                <span class="stat-label">TOTAL TIME</span>
              </div>
            </div>
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from "vue";

// --- AUDIO SYNTHESIS SYSTEM (Web Audio API) ---
class SoundSynth {
  constructor() {
    this.ctx = null;
    this.muted = false;
  }
  
  init() {
    if (this.muted) return;
    try {
      if (!this.ctx) {
        const AudioContextClass = window.AudioContext || window.webkitAudioContext;
        if (AudioContextClass) {
          this.ctx = new AudioContextClass();
        }
      }
      if (this.ctx && this.ctx.state === 'suspended') {
        this.ctx.resume();
      }
    } catch (e) {
      console.warn("Audio Context failed to initialize:", e);
    }
  }

  playMove() {
    this.init();
    if (this.muted || !this.ctx) return;
    try {
      const now = this.ctx.currentTime;
      const osc = this.ctx.createOscillator();
      const gain = this.ctx.createGain();
      osc.connect(gain);
      gain.connect(this.ctx.destination);
      
      osc.type = 'triangle';
      osc.frequency.setValueAtTime(140, now);
      osc.frequency.exponentialRampToValueAtTime(80, now + 0.04);
      
      gain.gain.setValueAtTime(0.015, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.04);
      
      osc.start(now);
      osc.stop(now + 0.04);
    } catch (e) {
      // Ignore
    }
  }

  playEat() {
    this.init();
    if (this.muted || !this.ctx) return;
    try {
      const now = this.ctx.currentTime;
      const osc = this.ctx.createOscillator();
      const gain = this.ctx.createGain();
      osc.connect(gain);
      gain.connect(this.ctx.destination);
      
      osc.type = 'sine';
      osc.frequency.setValueAtTime(400, now);
      osc.frequency.setValueAtTime(600, now + 0.06);
      osc.frequency.setValueAtTime(800, now + 0.12);
      
      gain.gain.setValueAtTime(0.04, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.2);
      
      osc.start(now);
      osc.stop(now + 0.2);
    } catch (e) {
      // Ignore
    }
  }

  playPowerup() {
    this.init();
    if (this.muted || !this.ctx) return;
    try {
      const now = this.ctx.currentTime;
      const osc = this.ctx.createOscillator();
      const gain = this.ctx.createGain();
      osc.connect(gain);
      gain.connect(this.ctx.destination);
      
      osc.type = 'triangle';
      osc.frequency.setValueAtTime(300, now);
      osc.frequency.linearRampToValueAtTime(900, now + 0.3);
      
      gain.gain.setValueAtTime(0.04, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.3);
      
      osc.start(now);
      osc.stop(now + 0.3);
    } catch (e) {
      // Ignore
    }
  }

  playShieldBreak() {
    this.init();
    if (this.muted || !this.ctx) return;
    try {
      const now = this.ctx.currentTime;
      const bufferSize = this.ctx.sampleRate * 0.25;
      const buffer = this.ctx.createBuffer(1, bufferSize, this.ctx.sampleRate);
      const data = buffer.getChannelData(0);
      for (let i = 0; i < bufferSize; i++) {
        data[i] = Math.random() * 2 - 1;
      }
      
      const noise = this.ctx.createBufferSource();
      noise.buffer = buffer;
      
      const filter = this.ctx.createBiquadFilter();
      filter.type = 'bandpass';
      filter.frequency.setValueAtTime(1000, now);
      filter.frequency.exponentialRampToValueAtTime(200, now + 0.25);
      
      const gain = this.ctx.createGain();
      gain.gain.setValueAtTime(0.08, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.25);
      
      noise.connect(filter);
      filter.connect(gain);
      gain.connect(this.ctx.destination);
      
      noise.start(now);
      noise.stop(now + 0.25);
    } catch (e) {
      // Ignore
    }
  }

  playGameOver() {
    this.init();
    if (this.muted || !this.ctx) return;
    try {
      const now = this.ctx.currentTime;
      const osc = this.ctx.createOscillator();
      const gain = this.ctx.createGain();
      osc.connect(gain);
      gain.connect(this.ctx.destination);
      
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(220, now);
      osc.frequency.linearRampToValueAtTime(55, now + 0.5);
      
      gain.gain.setValueAtTime(0.06, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.6);
      
      osc.start(now);
      osc.stop(now + 0.6);
    } catch (e) {
      // Ignore
    }
  }
}

// --- CONFIG & OPTIONS ---
const themes = {
  cyber: {
    bg: '#090d16',
    boardBg: '#0e1726',
    gridColor: 'rgba(0, 240, 255, 0.08)',
    glow: '#00f0ff',
    glowColor: 'rgba(0, 240, 255, 0.4)',
    textColor: '#ffffff'
  },
  sunset: {
    bg: '#150518',
    boardBg: '#230b28',
    gridColor: 'rgba(255, 0, 128, 0.08)',
    glow: '#ff0080',
    glowColor: 'rgba(255, 0, 128, 0.4)',
    textColor: '#ffffff'
  },
  matrix: {
    bg: '#000000',
    boardBg: '#050c05',
    gridColor: 'rgba(0, 255, 0, 0.12)',
    glow: '#00ff00',
    glowColor: 'rgba(0, 255, 0, 0.5)',
    textColor: '#00ff00'
  },
  chrono: {
    bg: '#141410',
    boardBg: '#20201a',
    gridColor: 'rgba(240, 190, 20, 0.08)',
    glow: '#f0be14',
    glowColor: 'rgba(240, 190, 20, 0.4)',
    textColor: '#ffffff'
  }
};

const skinOptions = [
  { id: 'viper', name: 'Cyber Viper', color: 'linear-gradient(135deg, #10b981, #047857)' },
  { id: 'cyberPink', name: 'Synthwave Pink', color: 'linear-gradient(135deg, #ff007f, #00f0ff)' },
  { id: 'gold', name: 'Imperial Gold', color: 'linear-gradient(135deg, #ffe066, #e6b800)' },
  { id: 'rainbow', name: 'Chroma Rainbow', color: 'linear-gradient(135deg, #ff0000, #ffff00, #00ff00, #0000ff, #8b00ff)' }
];

const themeOptions = [
  { id: 'cyber', name: 'Neon Cyber', colors: themes.cyber },
  { id: 'sunset', name: 'Sunset Drive', colors: themes.sunset },
  { id: 'matrix', name: 'Matrix Code', colors: themes.matrix },
  { id: 'chrono', name: 'Chrono Retro', colors: themes.chrono }
];

// --- APP STATE ---
const activeTab = ref('scores');
const score = ref(0);
const gameOver = ref(false);
const isPaused = ref(false);
const gameStarted = ref(false);
const isNewHighScore = ref(false);
const showDPad = ref(false);

const playerInitials = ref('');
const scoreSaved = ref(false);

// Preferences (loaded from LocalStorage)
const difficulty = ref(localStorage.getItem('snake_arcade_difficulty') || 'medium');
const wrapAround = ref(localStorage.getItem('snake_arcade_wraparound') !== 'false');
const soundMuted = ref(localStorage.getItem('snake_arcade_muted') === 'true');
const selectedSkin = ref(localStorage.getItem('snake_arcade_skin') || 'viper');
const selectedTheme = ref(localStorage.getItem('snake_arcade_theme') || 'cyber');

const highScore = ref(parseInt(localStorage.getItem('snake_arcade_high_score') || '0', 10));
const leaderboard = ref(JSON.parse(localStorage.getItem('snake_arcade_leaderboard') || '[]'));
const gameStats = ref(JSON.parse(localStorage.getItem('snake_arcade_stats') || JSON.stringify({
  gamesPlayed: 0,
  totalFoodEaten: 0,
  totalTimePlayed: 0
})));

const soundSynth = new SoundSynth();
soundSynth.muted = soundMuted.value;

// Watches
watch(difficulty, (n) => localStorage.setItem('snake_arcade_difficulty', n));
watch(wrapAround, (n) => localStorage.setItem('snake_arcade_wraparound', n.toString()));
watch(soundMuted, (n) => {
  localStorage.setItem('snake_arcade_muted', n.toString());
  soundSynth.muted = n;
});
watch(selectedSkin, (n) => localStorage.setItem('snake_arcade_skin', n));
watch(selectedTheme, (n) => localStorage.setItem('snake_arcade_theme', n));

const activeTheme = computed(() => themeOptions.find(t => t.id === selectedTheme.value) || themeOptions[0]);
const activeThemeColors = computed(() => activeTheme.value.colors);

// Leaderboard Qualification
const qualifiesForLeaderboard = computed(() => {
  if (score.value === 0) return false;
  if (leaderboard.value.length < 5) return true;
  return score.value > leaderboard.value[leaderboard.value.length - 1].score;
});

// --- CANVAS SETUP ---
const canvasRef = ref(null);
let ctx = null;

const gridWidth = 20;
const gridHeight = 20;
const cellSize = 30; // Canvas dimensions are 600x600 logical

// Game entities
const snake = ref([]);
const prevSnake = ref([]);
const direction = ref({ x: 1, y: 0 });
const nextDirection = ref({ x: 1, y: 0 });
const foods = ref([]);
const particles = ref([]);
const floatingTexts = ref([]);
const activePowerups = ref([]);

const shakeTime = ref(0);
let lastTickTime = 0;
let lastFrameTime = 0;
let animationFrameId = null;
let gameStartTime = 0;

const hasShield = computed(() => activePowerups.value.some(p => p.type === 'shield'));
const speedBoostActive = computed(() => activePowerups.value.some(p => p.type === 'speed'));
const slowMoActive = computed(() => activePowerups.value.some(p => p.type === 'slowmo'));

// --- CONTROLS ---
function initAudio() {
  soundSynth.init();
}

function setDifficulty(val) {
  difficulty.value = val;
}

watch(difficulty, () => {
  // If playing, reset speeds dynamically by updating ticker
  lastTickTime = performance.now();
});

function setWrapAround(val) {
  wrapAround.value = val;
}

function selectSkin(val) {
  selectedSkin.value = val;
}

function selectTheme(val) {
  selectedTheme.value = val;
}

function toggleMute() {
  soundMuted.value = !soundMuted.value;
}

function togglePause() {
  if (!gameStarted.value || gameOver.value) return;
  isPaused.value = !isPaused.value;
  soundSynth.init();
}

function resumeGame() {
  isPaused.value = false;
}

function quitToMenu() {
  gameStarted.value = false;
  gameOver.value = false;
  isPaused.value = false;
}

function changeDirection(newDir) {
  if (!gameStarted.value || gameOver.value || isPaused.value) return;
  
  const currentDir = direction.value;
  // Prevent 180-degree immediate reverse into body
  if (newDir.x !== 0 && currentDir.x !== 0) return;
  if (newDir.y !== 0 && currentDir.y !== 0) return;
  
  nextDirection.value = newDir;
  soundSynth.playMove();
}

// Swipes
let touchStartX = 0;
let touchStartY = 0;

function handleTouchStart(e) {
  touchStartX = e.touches[0].clientX;
  touchStartY = e.touches[0].clientY;
}

function handleTouchEnd(e) {
  if (!touchStartX || !touchStartY) return;
  
  const touchEndX = e.changedTouches[0].clientX;
  const touchEndY = e.changedTouches[0].clientY;
  
  const dx = touchEndX - touchStartX;
  const dy = touchEndY - touchStartY;
  const threshold = 35;
  
  if (Math.abs(dx) > Math.abs(dy)) {
    if (Math.abs(dx) > threshold) {
      if (dx > 0) changeDirection({ x: 1, y: 0 });
      else changeDirection({ x: -1, y: 0 });
    }
  } else {
    if (Math.abs(dy) > threshold) {
      if (dy > 0) changeDirection({ x: 0, y: 1 });
      else changeDirection({ x: 0, y: -1 });
    }
  }
  
  touchStartX = 0;
  touchStartY = 0;
}

function handleKey(e) {
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', ' '].includes(e.key)) {
    e.preventDefault();
  }
  
  if (e.key === ' ' || e.key === 'Spacebar') {
    if (!gameStarted.value) {
      startGame();
    } else if (gameOver.value) {
      startGame();
    } else {
      togglePause();
    }
    return;
  }
  
  if (isPaused.value || gameOver.value || !gameStarted.value) return;
  
  switch (e.key.toLowerCase()) {
    case 'w':
    case 'arrowup':
      changeDirection({ x: 0, y: -1 });
      break;
    case 's':
    case 'arrowdown':
      changeDirection({ x: 0, y: 1 });
      break;
    case 'a':
    case 'arrowleft':
      changeDirection({ x: -1, y: 0 });
      break;
    case 'd':
    case 'arrowright':
      changeDirection({ x: 1, y: 0 });
      break;
  }
}

// --- GAME LOGIC ENGINE ---
function startGame() {
  soundSynth.init();
  
  // Grid coordinates initial spawn
  snake.value = [
    { x: 10, y: 10 },
    { x: 9, y: 10 },
    { x: 8, y: 10 }
  ];
  prevSnake.value = JSON.parse(JSON.stringify(snake.value));
  
  direction.value = { x: 1, y: 0 };
  nextDirection.value = { x: 1, y: 0 };
  
  score.value = 0;
  activePowerups.value = [];
  foods.value = [];
  particles.value = [];
  floatingTexts.value = [];
  shakeTime.value = 0;
  
  spawnFood('normal');
  
  gameOver.value = false;
  isPaused.value = false;
  gameStarted.value = true;
  isNewHighScore.value = false;
  scoreSaved.value = false;
  playerInitials.value = '';
  
  gameStartTime = Date.now();
  lastTickTime = performance.now();
  lastFrameTime = performance.now();
}

function spawnFood(type = 'normal') {
  let attempts = 0;
  let rx, ry;
  const maxAttempts = 200;
  
  do {
    rx = Math.floor(Math.random() * gridWidth);
    ry = Math.floor(Math.random() * gridHeight);
    attempts++;
  } while (
    (snake.value.some(seg => seg.x === rx && seg.y === ry) || 
     foods.value.some(f => f.x === rx && f.y === ry)) && 
    attempts < maxAttempts
  );
  
  foods.value.push({ x: rx, y: ry, type });
}

function triggerScreenShake(ticks) {
  shakeTime.value = ticks;
}

function applyScreenShake(canvasCtx) {
  if (shakeTime.value > 0) {
    const dx = Math.random() * 8 - 4;
    const dy = Math.random() * 8 - 4;
    canvasCtx.translate(dx, dy);
    shakeTime.value--;
  }
}

function createExplosion(x, y, color) {
  const count = 15;
  const px = x * cellSize + cellSize / 2;
  const py = y * cellSize + cellSize / 2;
  
  for (let i = 0; i < count; i++) {
    const angle = Math.random() * Math.PI * 2;
    const speed = Math.random() * 4 + 2;
    particles.value.push({
      x: px,
      y: py,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      color: color,
      alpha: 1,
      size: Math.random() * 4 + 3
    });
  }
}

function createFloatingText(x, y, text, color) {
  const px = x * cellSize + cellSize / 2;
  const py = y * cellSize;
  floatingTexts.value.push({
    x: px,
    y: py,
    text: text,
    color: color,
    alpha: 1
  });
}

function consumeShield() {
  const idx = activePowerups.value.findIndex(p => p.type === 'shield');
  if (idx !== -1) {
    activePowerups.value.splice(idx, 1);
  }
  soundSynth.playShieldBreak();
  triggerScreenShake(12);
  const head = snake.value[0];
  createFloatingText(head.x, head.y, 'SHIELD BROKEN!', '#00f0ff');
}

function endGame() {
  gameOver.value = true;
  soundSynth.playGameOver();
  triggerScreenShake(20);
  
  // Aggregate stats
  gameStats.value.gamesPlayed++;
  const sessionDuration = Date.now() - gameStartTime;
  gameStats.value.totalTimePlayed += sessionDuration;
  localStorage.setItem('snake_arcade_stats', JSON.stringify(gameStats.value));
  
  // High scores update
  if (score.value > highScore.value) {
    highScore.value = score.value;
    localStorage.setItem('snake_arcade_high_score', highScore.value.toString());
    isNewHighScore.value = true;
  }
  
  // Automatically open leaderboard tab on gameover to show position
  activeTab.value = 'scores';
}

function submitScore() {
  if (!playerInitials.value) return;
  const name = playerInitials.value.trim().toUpperCase() || 'AAA';
  
  leaderboard.value.push({
    name,
    score: score.value,
    date: new Date().toLocaleDateString()
  });
  
  leaderboard.value.sort((a, b) => b.score - a.score);
  leaderboard.value = leaderboard.value.slice(0, 5);
  
  localStorage.setItem('snake_arcade_leaderboard', JSON.stringify(leaderboard.value));
  scoreSaved.value = true;
}

function updateGame() {
  if (gameOver.value || isPaused.value || !gameStarted.value) return;
  
  prevSnake.value = JSON.parse(JSON.stringify(snake.value));
  direction.value = nextDirection.value;
  
  const head = snake.value[0];
  let newHeadX = head.x + direction.value.x;
  let newHeadY = head.y + direction.value.y;
  
  // 1. Wall colliders
  if (newHeadX < 0 || newHeadX >= gridWidth || newHeadY < 0 || newHeadY >= gridHeight) {
    if (wrapAround.value) {
      newHeadX = (newHeadX + gridWidth) % gridWidth;
      newHeadY = (newHeadY + gridHeight) % gridHeight;
    } else {
      if (hasShield.value) {
        consumeShield();
        newHeadX = (newHeadX + gridWidth) % gridWidth;
        newHeadY = (newHeadY + gridHeight) % gridHeight;
      } else {
        endGame();
        return;
      }
    }
  }
  
  // 2. Self collisions
  const selfCollision = snake.value.slice(0, -1).some(seg => seg.x === newHeadX && seg.y === newHeadY);
  if (selfCollision) {
    if (hasShield.value) {
      consumeShield();
    } else {
      endGame();
      return;
    }
  }
  
  // Append new head
  snake.value.unshift({ x: newHeadX, y: newHeadY });
  
  // 3. Food colliders
  let foodIdx = -1;
  for (let i = 0; i < foods.value.length; i++) {
    if (foods.value[i].x === newHeadX && foods.value[i].y === newHeadY) {
      foodIdx = i;
      break;
    }
  }
  
  if (foodIdx !== -1) {
    const eatenFood = foods.value[foodIdx];
    foods.value.splice(foodIdx, 1);
    
    let points = 1;
    let textColor = '#ff2d55';
    
    const doublePoints = speedBoostActive.value;
    if (doublePoints) points = 2;
    
    if (eatenFood.type === 'normal') {
      soundSynth.playEat();
      createExplosion(eatenFood.x, eatenFood.y, '#ff2d55');
      createFloatingText(eatenFood.x, eatenFood.y, `+${points}`, '#ff2d55');
      
      gameStats.value.totalFoodEaten++;
      
      const hasPower = foods.value.some(f => f.type !== 'normal');
      if (!hasPower && Math.random() < 0.35) {
        const types = ['shield', 'speed', 'slowmo'];
        const randomType = types[Math.floor(Math.random() * types.length)];
        spawnFood(randomType);
      }
    } else {
      soundSynth.playPowerup();
      points = 2;
      
      if (eatenFood.type === 'shield') {
        textColor = '#00f0ff';
        createExplosion(eatenFood.x, eatenFood.y, '#00f0ff');
        createFloatingText(eatenFood.x, eatenFood.y, 'SHIELD CHARGED!', '#00f0ff');
        // Prevent stacking duplicate shields
        activePowerups.value = activePowerups.value.filter(p => p.type !== 'shield');
        activePowerups.value.push({ type: 'shield', duration: 999999, maxDuration: 1 });
      } else if (eatenFood.type === 'speed') {
        textColor = '#ffcc00';
        createExplosion(eatenFood.x, eatenFood.y, '#ffcc00');
        createFloatingText(eatenFood.x, eatenFood.y, '2X SPEED & SCORE!', '#ffcc00');
        activePowerups.value = activePowerups.value.filter(p => p.type !== 'speed');
        activePowerups.value.push({ type: 'speed', duration: 8000, maxDuration: 8000 });
      } else if (eatenFood.type === 'slowmo') {
        textColor = '#d400ff';
        createExplosion(eatenFood.x, eatenFood.y, '#d400ff');
        createFloatingText(eatenFood.x, eatenFood.y, 'SLOW MOTION!', '#d400ff');
        activePowerups.value = activePowerups.value.filter(p => p.type !== 'slowmo');
        activePowerups.value.push({ type: 'slowmo', duration: 8000, maxDuration: 8000 });
      }
    }
    
    score.value += points;
    
    const normalCount = foods.value.filter(f => f.type === 'normal').length;
    if (normalCount === 0) {
      spawnFood('normal');
    }
  } else {
    snake.value.pop();
  }
}

// --- DRAWING CANVAS ---
function getSnakeBodyStyle(canvasCtx, index, length, skin, time) {
  if (skin === 'viper') {
    return '#10b981';
  }
  if (skin === 'cyberPink') {
    const ratio = index / Math.max(1, length);
    const hue = 330 - (ratio * 150);
    return `hsl(${hue}, 100%, 60%)`;
  }
  if (skin === 'gold') {
    const ratio = index / Math.max(1, length);
    const l = 60 - ratio * 20;
    return `hsl(45, 100%, ${l}%)`;
  }
  if (skin === 'rainbow') {
    const hue = (index * 15 + time / 15) % 360;
    return `hsl(${hue}, 100%, 60%)`;
  }
  return '#10b981';
}

function strokeBodySegment(segPoints, startOffset, totalLength) {
  if (segPoints.length === 0) return;
  if (segPoints.length === 1) {
    ctx.beginPath();
    ctx.arc(segPoints[0].x, segPoints[0].y, cellSize * 0.4, 0, Math.PI * 2);
    ctx.fillStyle = getSnakeBodyStyle(ctx, startOffset, totalLength, selectedSkin.value, Date.now());
    ctx.fill();
    return;
  }
  
  ctx.beginPath();
  ctx.moveTo(segPoints[0].x, segPoints[0].y);
  for (let i = 1; i < segPoints.length; i++) {
    ctx.lineTo(segPoints[i].x, segPoints[i].y);
  }
  
  ctx.lineWidth = cellSize * 0.8;
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';
  ctx.strokeStyle = getSnakeBodyStyle(ctx, startOffset, totalLength, selectedSkin.value, Date.now());
  ctx.stroke();
}

function drawFoodItem(foodItem, time) {
  const fx = foodItem.x * cellSize + cellSize / 2;
  const fy = foodItem.y * cellSize + cellSize / 2;
  const radius = cellSize * 0.35;
  
  ctx.save();
  ctx.shadowBlur = 12;
  
  if (foodItem.type === 'normal') {
    ctx.fillStyle = '#ff2d55';
    ctx.shadowColor = '#ff2d55';
    ctx.beginPath();
    ctx.arc(fx, fy, radius, 0, Math.PI * 2);
    ctx.fill();
    
    ctx.strokeStyle = '#10b981';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(fx, fy - radius + 2);
    ctx.quadraticCurveTo(fx + 4, fy - radius - 5, fx + 7, fy - radius - 4);
    ctx.stroke();
  } else if (foodItem.type === 'shield') {
    ctx.fillStyle = '#00f0ff';
    ctx.shadowColor = '#00f0ff';
    
    const pulseFactor = Math.sin(time / 100) * 0.1 + 0.9;
    ctx.beginPath();
    ctx.arc(fx, fy, radius * pulseFactor * 0.8, 0, Math.PI * 2);
    ctx.fill();
    
    ctx.strokeStyle = 'rgba(0, 240, 255, 0.6)';
    ctx.lineWidth = 1.5;
    ctx.beginPath();
    ctx.arc(fx, fy, radius * 1.3, time / 200, time / 200 + Math.PI * 1.5);
    ctx.stroke();
  } else if (foodItem.type === 'speed') {
    ctx.fillStyle = '#ffcc00';
    ctx.shadowColor = '#ffcc00';
    
    ctx.translate(fx, fy);
    ctx.rotate(time / 150);
    
    ctx.beginPath();
    ctx.moveTo(0, -radius * 1.2);
    ctx.lineTo(radius * 0.8, 0);
    ctx.lineTo(0, radius * 1.2);
    ctx.lineTo(-radius * 0.8, 0);
    ctx.closePath();
    ctx.fill();
  } else if (foodItem.type === 'slowmo') {
    ctx.fillStyle = '#d400ff';
    ctx.shadowColor = '#d400ff';
    
    ctx.translate(fx, fy);
    ctx.rotate(Math.sin(time / 200) * 0.4);
    
    if (ctx.roundRect) {
      ctx.beginPath();
      ctx.roundRect(-radius * 0.5, -radius * 1.1, radius, radius * 2.2, radius * 0.5);
      ctx.fill();
    } else {
      ctx.beginPath();
      ctx.rect(-radius * 0.5, -radius * 1.1, radius, radius * 2.2);
      ctx.fill();
    }
  }
  
  ctx.restore();
}

function updateParticles() {
  for (let i = particles.value.length - 1; i >= 0; i--) {
    const p = particles.value[i];
    p.x += p.vx;
    p.y += p.vy;
    p.alpha -= 0.03;
    p.size *= 0.96;
    if (p.alpha <= 0 || p.size <= 0.5) {
      particles.value.splice(i, 1);
    }
  }
}

function drawParticles(canvasCtx) {
  canvasCtx.save();
  for (const p of particles.value) {
    canvasCtx.fillStyle = p.color;
    canvasCtx.globalAlpha = p.alpha;
    canvasCtx.shadowBlur = 8;
    canvasCtx.shadowColor = p.color;
    canvasCtx.beginPath();
    canvasCtx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
    canvasCtx.fill();
  }
  canvasCtx.restore();
}

function updateFloatingTexts() {
  for (let i = floatingTexts.value.length - 1; i >= 0; i--) {
    const t = floatingTexts.value[i];
    t.y -= 0.6;
    t.alpha -= 0.02;
    if (t.alpha <= 0) {
      floatingTexts.value.splice(i, 1);
    }
  }
}

function drawFloatingTexts(canvasCtx) {
  canvasCtx.save();
  canvasCtx.font = 'bold 15px "Space Grotesk"';
  canvasCtx.textAlign = 'center';
  for (const t of floatingTexts.value) {
    canvasCtx.fillStyle = t.color;
    canvasCtx.globalAlpha = Math.max(0, t.alpha);
    canvasCtx.shadowBlur = 4;
    canvasCtx.shadowColor = t.color;
    canvasCtx.fillText(t.text, t.x, t.y);
  }
  canvasCtx.restore();
}

function drawCanvas(timestamp, progress = 1) {
  if (!ctx || !canvasRef.value) return;
  
  ctx.save();
  
  // Render Board Bg
  ctx.fillStyle = activeThemeColors.value.boardBg;
  ctx.fillRect(0, 0, canvasRef.value.width, canvasRef.value.height);
  
  // Camera Shake
  applyScreenShake(ctx);
  
  // Render grid lines
  ctx.strokeStyle = activeThemeColors.value.gridColor;
  ctx.lineWidth = 0.5;
  for (let i = 0; i <= gridWidth; i++) {
    ctx.beginPath();
    ctx.moveTo(i * cellSize, 0);
    ctx.lineTo(i * cellSize, canvasRef.value.height);
    ctx.stroke();
    
    ctx.beginPath();
    ctx.moveTo(0, i * cellSize);
    ctx.lineTo(canvasRef.value.width, i * cellSize);
    ctx.stroke();
  }
  
  // Render foods
  foods.value.forEach(f => drawFoodItem(f, timestamp));
  
  // Update and draw particles
  updateParticles();
  drawParticles(ctx);
  
  // Update and draw floating texts
  updateFloatingTexts();
  drawFloatingTexts(ctx);
  
  // Render Snake
  if (snake.value.length > 0) {
    const points = [];
    const len = snake.value.length;
    const prevLen = prevSnake.value.length;
    
    for (let i = 0; i < len; i++) {
      if (i < prevLen) {
        let px = prevSnake.value[i].x;
        let py = prevSnake.value[i].y;
        let cx = snake.value[i].x;
        let cy = snake.value[i].y;
        
        if (cx - px < -1.5) px -= gridWidth;
        else if (cx - px > 1.5) px += gridWidth;
        
        if (cy - py < -1.5) py -= gridHeight;
        else if (cy - py > 1.5) py += gridHeight;
        
        const interpX = px + (cx - px) * progress;
        const interpY = py + (cy - py) * progress;
        
        points.push({
          x: interpX * cellSize + cellSize / 2,
          y: interpY * cellSize + cellSize / 2
        });
      } else {
        points.push({
          x: snake.value[i].x * cellSize + cellSize / 2,
          y: snake.value[i].y * cellSize + cellSize / 2
        });
      }
    }
    
    ctx.save();
    ctx.shadowBlur = 10;
    ctx.shadowColor = activeThemeColors.value.glow;
    
    let startIdx = 0;
    for (let i = 1; i < points.length; i++) {
      const p1 = points[i - 1];
      const p2 = points[i];
      const dist = Math.hypot(p2.x - p1.x, p2.y - p1.y);
      
      if (dist > cellSize * 2) {
        strokeBodySegment(points.slice(startIdx, i), startIdx, len);
        startIdx = i;
      }
    }
    if (startIdx < points.length) {
      strokeBodySegment(points.slice(startIdx), startIdx, len);
    }
    ctx.restore();
    
    // Render head eyes
    if (points.length > 0) {
      const head = points[0];
      const dirAngle = Math.atan2(direction.value.y, direction.value.x);
      const eyeOffsetAngle = 0.55;
      const eyeDist = cellSize * 0.25;
      
      const leftEyeX = head.x + Math.cos(dirAngle - eyeOffsetAngle) * eyeDist;
      const leftEyeY = head.y + Math.sin(dirAngle - eyeOffsetAngle) * eyeDist;
      
      const rightEyeX = head.x + Math.cos(dirAngle + eyeOffsetAngle) * eyeDist;
      const rightEyeY = head.y + Math.sin(dirAngle + eyeOffsetAngle) * eyeDist;
      
      // Eye whites
      ctx.fillStyle = '#ffffff';
      ctx.beginPath();
      ctx.arc(leftEyeX, leftEyeY, cellSize * 0.15, 0, Math.PI * 2);
      ctx.arc(rightEyeX, rightEyeY, cellSize * 0.15, 0, Math.PI * 2);
      ctx.fill();
      
      // Eye tracking pupils look towards closest food item
      let lookAngle = dirAngle;
      if (foods.value.length > 0) {
        const nearestFood = foods.value[0];
        lookAngle = Math.atan2(nearestFood.y * cellSize + cellSize / 2 - head.y, nearestFood.x * cellSize + cellSize / 2 - head.x);
      }
      
      const pupilDist = cellSize * 0.05;
      const leftPupilX = leftEyeX + Math.cos(lookAngle) * pupilDist;
      const leftPupilY = leftEyeY + Math.sin(lookAngle) * pupilDist;
      const rightPupilX = rightEyeX + Math.cos(lookAngle) * pupilDist;
      const rightPupilY = rightEyeY + Math.sin(lookAngle) * pupilDist;
      
      ctx.fillStyle = '#000000';
      ctx.beginPath();
      ctx.arc(leftPupilX, leftPupilY, cellSize * 0.075, 0, Math.PI * 2);
      ctx.arc(rightPupilX, rightPupilY, cellSize * 0.075, 0, Math.PI * 2);
      ctx.fill();
      
      // Gold Skin Crown
      if (selectedSkin.value === 'gold') {
        ctx.save();
        ctx.translate(head.x, head.y);
        ctx.rotate(dirAngle + Math.PI / 2);
        
        ctx.fillStyle = '#f59e0b';
        ctx.strokeStyle = '#ffffff';
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.moveTo(-8, -8);
        ctx.lineTo(-12, -20);
        ctx.lineTo(-4, -14);
        ctx.lineTo(0, -23);
        ctx.lineTo(4, -14);
        ctx.lineTo(12, -20);
        ctx.lineTo(8, -8);
        ctx.closePath();
        ctx.fill();
        ctx.stroke();
        ctx.restore();
      }
    }
  }
  
  ctx.restore();
}

function gameLoop(timestamp) {
  // Compute Frame time delta for powerups timer
  if (!lastFrameTime) lastFrameTime = timestamp;
  const frameDt = timestamp - lastFrameTime;
  lastFrameTime = timestamp;
  
  // Decrement powerup timer durations
  if (gameStarted.value && !gameOver.value && !isPaused.value) {
    for (let i = activePowerups.value.length - 1; i >= 0; i--) {
      const p = activePowerups.value[i];
      if (p.type !== 'shield') {
        p.duration -= frameDt;
        if (p.duration <= 0) {
          activePowerups.value.splice(i, 1);
        }
      }
    }
  }
  
  if (!gameStarted.value || gameOver.value || isPaused.value) {
    drawCanvas(timestamp);
    animationFrameId = requestAnimationFrame(gameLoop);
    return;
  }
  
  if (!lastTickTime) lastTickTime = timestamp;
  
  // Select tick speed based on difficulty and active powerups
  let baseSpeed = 120;
  if (difficulty.value === 'easy') baseSpeed = 175;
  else if (difficulty.value === 'hard') baseSpeed = 75;
  
  let currentSpeed = baseSpeed;
  if (speedBoostActive.value) {
    currentSpeed = baseSpeed * 0.6;
  } else if (slowMoActive.value) {
    currentSpeed = baseSpeed * 1.5;
  }
  
  const elapsed = timestamp - lastTickTime;
  
  if (elapsed >= currentSpeed) {
    updateGame();
    // Safety check for tab hibernation spikes
    if (elapsed > currentSpeed * 3) {
      lastTickTime = timestamp;
    } else {
      lastTickTime = timestamp - (elapsed % currentSpeed);
    }
  }
  
  // Compute interpolation progress
  const progress = Math.min(1, elapsed / currentSpeed);
  drawCanvas(timestamp, progress);
  
  animationFrameId = requestAnimationFrame(gameLoop);
}

// Format MS to string MM:SS
function formatTime(ms) {
  const totalSeconds = Math.floor(ms / 1000);
  const minutes = Math.floor(totalSeconds / 60);
  const seconds = totalSeconds % 60;
  return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
}

// --- SYSTEM HOOKS ---
onMounted(() => {
  ctx = canvasRef.value.getContext('2d');
  
  window.addEventListener('keydown', handleKey);
  animationFrameId = requestAnimationFrame(gameLoop);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKey);
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
});
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;800&family=Space+Grotesk:wght@400;700&display=swap');

:root {
  /* Neon cyber theme */
  --cyber-bg: #090d16;
  --cyber-panel: rgba(13, 22, 38, 0.75);
  --cyber-glow: #00f0ff;
  --cyber-accent: #ff007f;
  --cyber-text: #ffffff;
  
  /* Sunset drive theme */
  --sunset-bg: #150518;
  --sunset-panel: rgba(30, 10, 35, 0.75);
  --sunset-glow: #ff0080;
  --sunset-accent: #ffb800;
  --sunset-text: #ffffff;
  
  /* Matrix theme */
  --matrix-bg: #000000;
  --matrix-panel: rgba(5, 12, 5, 0.85);
  --matrix-glow: #00ff00;
  --matrix-accent: #008800;
  --matrix-text: #00ff00;
  
  /* Chrono Retro theme */
  --chrono-bg: #141410;
  --chrono-panel: rgba(32, 32, 26, 0.8);
  --chrono-glow: #f0be14;
  --chrono-accent: #d2a00c;
  --chrono-text: #ffffff;
}

body {
  margin: 0;
  padding: 0;
  background-color: #030712;
  color: #f3f4f6;
  font-family: 'Outfit', sans-serif;
  overflow-x: hidden;
}

.arcade-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  min-height: 100vh;
  padding: 2rem 1rem;
  box-sizing: border-box;
  transition: all 0.5s ease;
}

.arcade-container.cyber {
  background: radial-gradient(circle at center, #0f1c3f 0%, var(--cyber-bg) 100%);
  --theme-glow: var(--cyber-glow);
  --theme-accent: var(--cyber-accent);
  --theme-panel: var(--cyber-panel);
  --theme-text: var(--cyber-text);
}

.arcade-container.sunset {
  background: radial-gradient(circle at center, #2e0828 0%, var(--sunset-bg) 100%);
  --theme-glow: var(--sunset-glow);
  --theme-accent: var(--sunset-accent);
  --theme-panel: var(--sunset-panel);
  --theme-text: var(--sunset-text);
}

.arcade-container.matrix {
  background: radial-gradient(circle at center, #0a1f0a 0%, var(--matrix-bg) 100%);
  --theme-glow: var(--matrix-glow);
  --theme-accent: var(--matrix-accent);
  --theme-panel: var(--matrix-panel);
  --theme-text: var(--matrix-text);
}

.arcade-container.chrono {
  background: radial-gradient(circle at center, #24241d 0%, var(--chrono-bg) 100%);
  --theme-glow: var(--chrono-glow);
  --theme-accent: var(--chrono-accent);
  --theme-panel: var(--chrono-panel);
  --theme-text: var(--chrono-text);
}

.arcade-header {
  text-align: center;
  margin-bottom: 2rem;
}

.logo-text {
  font-family: 'Space Grotesk', sans-serif;
  font-size: 3.5rem;
  font-weight: 800;
  margin: 0;
  letter-spacing: -2px;
  text-shadow: 0 0 10px var(--theme-glow), 0 0 20px var(--theme-glow);
  color: var(--theme-text);
}

.accent-text {
  color: var(--theme-accent);
  text-shadow: 0 0 10px var(--theme-accent), 0 0 20px var(--theme-accent);
}

.subtitle-text {
  font-size: 0.85rem;
  font-weight: 600;
  letter-spacing: 5px;
  color: rgba(255, 255, 255, 0.4);
  margin-top: 0.2rem;
}

.arcade-main {
  display: grid;
  grid-template-columns: 1.2fr 0.8fr;
  gap: 2.5rem;
  width: 100%;
  max-width: 1050px;
}

.game-section {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.hud-container {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
}

.hud-stat {
  flex: 1;
  background: var(--theme-panel);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  padding: 0.75rem 1.25rem;
  display: flex;
  flex-direction: column;
  backdrop-filter: blur(12px);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

.hud-label {
  font-size: 0.75rem;
  color: rgba(255, 255, 255, 0.4);
  font-weight: 600;
  letter-spacing: 1px;
}

.hud-value {
  font-family: 'Space Grotesk', sans-serif;
  font-size: 2rem;
  font-weight: 700;
  color: var(--theme-glow);
  text-shadow: 0 0 10px var(--theme-glow);
}

.board-wrapper {
  position: relative;
  width: 100%;
  aspect-ratio: 1/1;
  border-radius: 16px;
  overflow: hidden;
  border: 2px solid var(--theme-glow);
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.6), 0 0 15px rgba(255, 255, 255, 0.05);
}

.game-canvas {
  width: 100%;
  height: 100%;
  display: block;
}

.screen-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(8px);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  box-sizing: border-box;
  text-align: center;
  z-index: 10;
}

.screen-title {
  font-family: 'Space Grotesk', sans-serif;
  font-size: 3rem;
  font-weight: 700;
  color: var(--theme-glow);
  margin: 0 0 0.5rem 0;
  text-shadow: 0 0 15px var(--theme-glow);
}

.screen-title.game-over-text {
  color: var(--theme-accent);
  text-shadow: 0 0 15px var(--theme-accent);
}

.screen-subtitle {
  font-size: 1.1rem;
  color: rgba(255, 255, 255, 0.6);
  margin-bottom: 2rem;
}

.screen-controls-hint {
  font-size: 0.8rem;
  color: rgba(255, 255, 255, 0.35);
  margin-top: 1.5rem;
}

.pulse-glow {
  animation: pulse-glow-anim 2s infinite ease-in-out;
}

@keyframes pulse-glow-anim {
  0%, 100% {
    opacity: 0.8;
    text-shadow: 0 0 10px var(--theme-glow);
  }
  50% {
    opacity: 1;
    text-shadow: 0 0 25px var(--theme-glow), 0 0 40px var(--theme-glow);
  }
}

.arcade-btn {
  background: linear-gradient(135deg, var(--theme-glow) 0%, rgba(255, 255, 255, 0) 100%);
  background-color: rgba(255, 255, 255, 0.05);
  color: #fff;
  border: 1px solid var(--theme-glow);
  padding: 0.8rem 2rem;
  border-radius: 8px;
  font-size: 1.1rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  letter-spacing: 1px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3), inset 0 0 10px rgba(255, 255, 255, 0.05);
}

.arcade-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 25px var(--theme-glow), inset 0 0 15px rgba(255, 255, 255, 0.1);
  background-color: var(--theme-glow);
  color: #000;
}

.arcade-btn:active {
  transform: translateY(1px);
}

.arcade-btn.danger {
  border-color: var(--theme-accent);
  background: linear-gradient(135deg, var(--theme-accent) 0%, rgba(0,0,0,0) 100%);
}

.arcade-btn.danger:hover {
  background-color: var(--theme-accent);
  color: #000;
  box-shadow: 0 6px 25px var(--theme-accent);
}

.arcade-btn.mini {
  padding: 0.5rem 1rem;
  font-size: 0.85rem;
  border-radius: 6px;
}

.arcade-btn.mini.active {
  background: var(--theme-glow);
  color: #000;
  box-shadow: 0 0 15px var(--theme-glow);
}

.btn-group {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.5rem;
}

.overlay-buttons {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  width: 220px;
}

.run-stats {
  margin-bottom: 1.5rem;
  font-size: 1.2rem;
}

.run-stats .highlight {
  color: var(--theme-glow);
  font-weight: 700;
  font-family: 'Space Grotesk', sans-serif;
  font-size: 1.5rem;
}

.new-record {
  color: var(--theme-accent);
  font-weight: 700;
  margin-top: 0.5rem;
}

.leaderboard-entry {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  padding: 1.25rem;
  border-radius: 12px;
  margin-bottom: 1.5rem;
  width: 100%;
  max-width: 320px;
}

.entry-row {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.75rem;
  justify-content: center;
}

.initials-input {
  background: rgba(0, 0, 0, 0.5);
  border: 1px solid var(--theme-glow);
  color: #fff;
  font-family: 'Space Grotesk', sans-serif;
  font-size: 1.2rem;
  padding: 0.5rem;
  border-radius: 6px;
  text-align: center;
  width: 80px;
  text-transform: uppercase;
}

.initials-input:focus {
  outline: none;
  box-shadow: 0 0 10px var(--theme-glow);
}

.powerups-hud {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.powerup-badge {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 8px;
  padding: 0.5rem 0.75rem;
}

.powerup-badge.shield { border-color: #00f0ff; color: #00f0ff; }
.powerup-badge.speed { border-color: #ffcc00; color: #ffcc00; }
.powerup-badge.slowmo { border-color: #d400ff; color: #d400ff; }

.powerup-icon {
  display: flex;
  align-items: center;
}

.powerup-name {
  font-size: 0.75rem;
  font-weight: 700;
  width: 70px;
}

.powerup-progress {
  flex: 1;
  height: 4px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 2px;
  overflow: hidden;
}

.powerup-bar {
  height: 100%;
  border-radius: 2px;
  transition: width 0.1s linear;
}

.shield .powerup-bar { background-color: #00f0ff; }
.speed .powerup-bar { background-color: #ffcc00; }
.slowmo .powerup-bar { background-color: #d400ff; }

.dashboard-section {
  background: var(--theme-panel);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 16px;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  backdrop-filter: blur(12px);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
}

.dashboard-tabs {
  display: flex;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  margin-bottom: 1.5rem;
  gap: 0.5rem;
}

.tab-btn {
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.4);
  padding: 0.75rem 1rem;
  font-size: 0.85rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.3s;
  position: relative;
  letter-spacing: 1px;
}

.tab-btn:hover {
  color: rgba(255, 255, 255, 0.8);
}

.tab-btn.active {
  color: var(--theme-glow);
  text-shadow: 0 0 10px var(--theme-glow);
}

.tab-btn.active::after {
  content: '';
  position: absolute;
  bottom: -1px;
  left: 0;
  width: 100%;
  height: 2px;
  background-color: var(--theme-glow);
  box-shadow: 0 0 10px var(--theme-glow);
}

.tab-content {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  flex: 1;
}

.setting-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.setting-group h3 {
  margin: 0;
  font-size: 0.85rem;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.4);
  letter-spacing: 1px;
}

.setting-desc {
  font-size: 0.8rem;
  color: rgba(255, 255, 255, 0.4);
  line-height: 1.4;
  margin: 0.2rem 0 0 0;
}

.skins-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.skin-card {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  padding: 1rem;
  display: flex;
  align-items: center;
  gap: 0.75rem;
  cursor: pointer;
  transition: all 0.3s;
}

.skin-card:hover {
  background: rgba(255, 255, 255, 0.08);
  border-color: rgba(255, 255, 255, 0.2);
}

.skin-card.active {
  border-color: var(--theme-glow);
  background: rgba(255, 255, 255, 0.05);
  box-shadow: 0 0 15px rgba(0, 240, 255, 0.1);
}

.skin-preview {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  box-shadow: 0 0 8px rgba(0,0,0,0.5);
}

.skin-name {
  font-size: 0.85rem;
  font-weight: 600;
}

.themes-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.theme-card {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  padding: 1rem;
  display: flex;
  align-items: center;
  gap: 0.75rem;
  cursor: pointer;
  transition: all 0.3s;
}

.theme-card:hover {
  background: rgba(255, 255, 255, 0.08);
  border-color: rgba(255, 255, 255, 0.2);
}

.theme-card.active {
  border-color: var(--theme-glow);
  background: rgba(255, 255, 255, 0.05);
  box-shadow: 0 0 15px rgba(0, 240, 255, 0.1);
}

.theme-colors-preview {
  display: flex;
  gap: 4px;
}

.theme-color {
  width: 12px;
  height: 12px;
  border-radius: 50%;
}

.theme-name {
  font-size: 0.85rem;
  font-weight: 600;
}

.leaderboard-table {
  display: flex;
  flex-direction: column;
  background: rgba(0, 0, 0, 0.2);
  border-radius: 12px;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.table-header {
  display: grid;
  grid-template-columns: 0.8fr 1.2fr 1fr;
  padding: 0.75rem 1rem;
  background: rgba(255, 255, 255, 0.04);
  font-size: 0.75rem;
  font-weight: 700;
  color: rgba(255, 255, 255, 0.4);
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
  letter-spacing: 1px;
}

.table-row {
  display: grid;
  grid-template-columns: 0.8fr 1.2fr 1fr;
  padding: 0.75rem 1rem;
  font-size: 0.9rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.03);
}

.table-row:last-child {
  border-bottom: none;
}

.table-row.gold { color: #ffe066; font-weight: 700; }
.table-row.silver { color: #e2e8f0; font-weight: 700; }
.table-row.bronze { color: #d97706; font-weight: 700; }

.rank-col {
  display: flex;
  align-items: center;
}

.initials-col {
  font-family: 'Space Grotesk', sans-serif;
  letter-spacing: 1px;
  font-weight: 600;
}

.score-col {
  font-family: 'Space Grotesk', sans-serif;
  font-weight: 700;
  text-align: right;
}

.no-records {
  padding: 2rem;
  text-align: center;
  font-size: 0.8rem;
  color: rgba(255, 255, 255, 0.3);
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.75rem;
}

.stat-box {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.05);
  padding: 1rem 0.5rem;
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

.stat-value {
  font-family: 'Space Grotesk', sans-serif;
  font-size: 1.4rem;
  font-weight: 700;
  color: var(--theme-glow);
}

.stat-label {
  font-size: 0.65rem;
  color: rgba(255, 255, 255, 0.4);
  font-weight: 600;
  letter-spacing: 0.5px;
  margin-top: 0.25rem;
}

.mobile-controls-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 1rem;
}

.toggle-dpad-btn {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.6);
  padding: 0.5rem 1rem;
  border-radius: 20px;
  font-size: 0.8rem;
  cursor: pointer;
  transition: all 0.3s;
}

.toggle-dpad-btn:hover {
  background: rgba(255, 255, 255, 0.08);
  color: #fff;
}

.dpad-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  margin-top: 1rem;
  background: rgba(0, 0, 0, 0.3);
  padding: 1rem;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.dpad-row {
  display: flex;
  gap: 5px;
}

.dpad-btn {
  width: 50px;
  height: 50px;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: #fff;
  border-radius: 8px;
  font-size: 1.2rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
  box-shadow: 0 4px 6px rgba(0,0,0,0.2);
}

.dpad-btn:hover {
  background: var(--theme-glow);
  color: #000;
  border-color: var(--theme-glow);
  box-shadow: 0 0 10px var(--theme-glow);
}

.dpad-btn:active {
  transform: scale(0.95);
}

.dpad-center {
  width: 50px;
  height: 50px;
}

@media (max-width: 900px) {
  .arcade-main {
    grid-template-columns: 1fr;
    gap: 2rem;
  }
  
  .logo-text {
    font-size: 2.8rem;
  }
}
</style>