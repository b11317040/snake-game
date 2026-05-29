<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type Position = { x: number; y: number }
type Bomb = Position & { id: number; explodeAt: number }
type Explosion = Position & { id: number; endAt: number }
type GameStatus = 'ready' | 'playing' | 'paused' | 'over' | 'reward'
type Language = 'zhTw' | 'en' | 'ja' | 'ko' | 'es' | 'fr' | 'de' | 'pt' | 'ru' | 'ar' | 'hi' | 'id' | 'vi' | 'th'
type RewardKey = 'life1' | 'life2' | 'shorten' | 'removeObstacle' | 'clearBomb' | 'slow' | 'shield' | 'hideObstacle'
type SnakeSkin = 'neon' | 'snake' | 'dragon' | 'robot' | 'cat' | 'avatar'
type GameMode = 'classic' | 'portal' | 'time' | 'treasure' | 'survival'

const boardSize = 20
const skillDurationMs = 10000
const respawnProtectionMs = 900
const timeModeStartMs = 60000
const explosionDurationMs = 850

const snake = ref<Position[]>([{ x: 10, y: 10 }])
const food = ref<Position>({ x: 5, y: 5 })
const goldenFood = ref<Position | null>(null)
const obstacles = ref<Position[]>([])
const bombs = ref<Bomb[]>([])
const explosions = ref<Explosion[]>([])

const direction = ref<Position>({ x: 1, y: 0 })
const nextDirection = ref<Position>({ x: 1, y: 0 })

const score = ref(0)
const bestScore = ref(0)
const lives = ref(1)
const level = ref(1)
const speed = ref(140)
const status = ref<GameStatus>('ready')
const language = ref<Language>('zhTw')
const snakeSkin = ref<SnakeSkin>('neon')
const gameMode = ref<GameMode>('classic')

const lastDifficultyLevel = ref(0)
const lastRewardScore = ref(0)

const currentSkillKey = ref<RewardKey | null>(null)
const rewardKey = ref<RewardKey | null>(null)

const shieldUntil = ref(0)
const slowUntil = ref(0)
const hideObstacleUntil = ref(0)
const invincibleUntil = ref(0)
const goldenFoodUntil = ref(0)
const timeAttackUntil = ref(0)
const survivalNextPointAt = ref(0)

const now = ref(Date.now())

const avatarImage = ref('')
const cameraMessage = ref('')
const cameraActive = ref(false)
const avatarMenuOpen = ref(false)
const videoRef = ref<HTMLVideoElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)

let cameraStream: MediaStream | null = null
let timer: number | undefined
let clockTimer: number | undefined
let bombIdSeed = 1
let explosionIdSeed = 1
let wasSlowActive = false

const languageOptions = [
  { value: 'zhTw', label: '繁體中文' },
  { value: 'en', label: 'English' },
  { value: 'ja', label: '日本語' },
  { value: 'ko', label: '한국어' },
  { value: 'es', label: 'Español' },
  { value: 'fr', label: 'Français' },
  { value: 'de', label: 'Deutsch' },
  { value: 'pt', label: 'Português' },
  { value: 'ru', label: 'Русский' },
  { value: 'ar', label: 'العربية' },
  { value: 'hi', label: 'हिन्दी' },
  { value: 'id', label: 'Bahasa Indonesia' },
  { value: 'vi', label: 'Tiếng Việt' },
  { value: 'th', label: 'ไทย' },
] as const

const snakeSkinOptions = [
  { value: 'neon', icon: '🟩' },
  { value: 'snake', icon: '🐍' },
  { value: 'dragon', icon: '🐉' },
  { value: 'robot', icon: '🤖' },
  { value: 'cat', icon: '😺' },
  { value: 'avatar', icon: '📸' },
] as const

const modeOptions = [
  { value: 'classic', icon: '🎮' },
  { value: 'portal', icon: '🌀' },
  { value: 'time', icon: '⏱️' },
  { value: 'treasure', icon: '💎' },
  { value: 'survival', icon: '🔥' },
] as const

const zhText = {
  project: 'Vue 專案',
  title: '貪食蛇遊戲',
  description: '選擇蛇的樣式與玩法模式，吃食物、躲炸彈、避開障礙，挑戰最高分。',
  language: '切換語言',
  snakeStyle: '蛇的樣式',
  mode: '遊戲模式',

  classic: '經典模式',
  portal: '穿越模式',
  time: '限時模式',
  treasure: '寶藏模式',
  survival: '生存模式',

  classicDesc: '標準玩法，撞牆或撞到危險物會扣生命。',
  portalDesc: '撞到牆不會死，會從地圖另一邊穿出來。',
  timeDesc: '60 秒限時挑戰，吃食物會增加時間。',
  treasureDesc: '會出現金色食物，吃到可以獲得額外分數。',
  survivalDesc: '分數會隨時間增加，但障礙與炸彈會越來越多。',

  neon: '霓虹方塊',
  snake: '小蛇',
  dragon: '龍',
  robot: '機器人',
  cat: '貓咪',
  avatar: '我的頭貼',

  status: '狀態',
  ready: '準備中',
  playing: '遊戲中',
  paused: '暫停中',
  over: '遊戲結束',
  reward: '技能輪盤',

  boardTitle: '20 × 20 遊戲地圖',
  score: '目前分數',
  finalScore: '本次分數',
  bestScore: '最高分數',
  lives: '生命',
  level: '關卡',
  speed: '蛇速',
  skill: '目前技能',
  nextReward: '下一次輪盤',
  obstacles: '障礙物',
  bombs: '炸彈',
  timeLeft: '剩餘時間',
  nextSurvivalPoint: '生存加分',
  modeFeature: '模式特色',

  avatarMenu: '自訂頭貼',
  cameraTitle: '拍照設定蛇頭',
  openCamera: '開啟鏡頭',
  capturePhoto: '拍照使用',
  closeCamera: '關閉鏡頭',
  clearAvatar: '清除頭貼',
  closeMenu: '關閉選單',
  cameraHint: '拍照後會自動切換成「我的頭貼」樣式，照片會以圓形顯示在蛇頭。',
  cameraReady: '鏡頭已開啟，可以拍照。',
  cameraDenied: '鏡頭無法開啟，可能是瀏覽器權限被拒絕，仍可使用其他蛇樣式。',
  cameraCaptured: '已套用圓形頭貼蛇頭。',

  legend: '圖示說明',
  noSkill: '無',
  allSkills: '可能出現的技能',

  start: '開始',
  pause: '暫停',
  resume: '繼續',
  restart: '重新開始',
  exit: '退出',

  readyMessage: '選好模式和蛇的樣式後，按開始即可遊玩。',
  pausedMessage: '遊戲已暫停，按繼續或空白鍵恢復。',
  overMessage: '遊戲結束！可以重新開始挑戰。',
  gameOverTitle: '遊戲結束',
  gameOverSubtitle: '這次挑戰結束了，再來一次刷新最高分！',

  rewardTitle: '技能輪盤獎勵',
  rewardButton: '領取並繼續',
  skillEffect: '技能效果',
  howToUse: '使用方式',

  controls: '操作方式',
  space: '空白鍵：暫停 / 繼續',

  rules: '遊戲規則',
  ruleFood: '吃到 🍎 食物 = +1 分；寶藏模式的 💎 金色食物 = +3 分。',
  ruleHazard: '撞牆、撞自己、撞 🧱 障礙、踩到 💣 炸彈或被 💥 爆炸波及會扣 1 顆生命。',
  ruleBomb: '💣 炸彈會不定時引爆，爆炸範圍是周圍 3×3 區域。',
  ruleLevel: '每 5 分進到下一關，速度會變快，障礙與炸彈也會增加。',
  ruleReward: '每 10 分會觸發技能輪盤，技能會自動生效。',
  ruleMode: '不同模式不是單純改難度，而是會改變遊戲玩法。',

  activeEffects: '技能倒數',
  noActiveEffect: '目前沒有技能效果',

  legendFood: '食物',
  legendGoldFood: '金色食物',
  legendObstacle: '障礙',
  legendBomb: '炸彈',
  legendExplosion: '爆炸範圍',

  secondPerStep: '秒 / 格',
  second: '秒',
}

const enText = {
  ...zhText,
  project: 'Vue Project',
  title: 'Snake Game',
  description: 'Choose a snake style and game mode, eat food, dodge bombs, avoid obstacles, and chase the best score.',
  language: 'Language',
  snakeStyle: 'Snake Style',
  mode: 'Game Mode',

  classic: 'Classic',
  portal: 'Portal',
  time: 'Time Attack',
  treasure: 'Treasure',
  survival: 'Survival',

  classicDesc: 'Standard snake gameplay. Walls and hazards cost lives.',
  portalDesc: 'Walls become portals. Going out from one side brings you back from the other side.',
  timeDesc: 'A 60-second challenge. Eating food adds time.',
  treasureDesc: 'Golden food appears and gives bonus points.',
  survivalDesc: 'Score increases over time, but hazards keep growing.',

  neon: 'Neon Block',
  snake: 'Snake',
  dragon: 'Dragon',
  robot: 'Robot',
  cat: 'Cat',
  avatar: 'My Avatar',

  status: 'Status',
  ready: 'Ready',
  playing: 'Playing',
  paused: 'Paused',
  over: 'Game Over',
  reward: 'Reward Wheel',

  boardTitle: '20 × 20 Game Board',
  score: 'Score',
  finalScore: 'Final Score',
  bestScore: 'Best Score',
  lives: 'Lives',
  level: 'Level',
  speed: 'Speed',
  skill: 'Current Skill',
  nextReward: 'Next Reward',
  obstacles: 'Obstacles',
  bombs: 'Bombs',
  timeLeft: 'Time Left',
  nextSurvivalPoint: 'Survival Point',
  modeFeature: 'Mode Feature',

  avatarMenu: 'Custom Avatar',
  cameraTitle: 'Take Photo for Snake Head',
  openCamera: 'Open Camera',
  capturePhoto: 'Capture',
  closeCamera: 'Close Camera',
  clearAvatar: 'Clear Avatar',
  closeMenu: 'Close Menu',
  cameraHint: 'After taking a photo, it will automatically switch to “My Avatar”. The photo is shown as a circular snake head.',
  cameraReady: 'Camera is ready.',
  cameraDenied: 'Camera could not be opened. Permission may be denied. Other snake styles still work.',
  cameraCaptured: 'Circular avatar snake head applied.',

  legend: 'Legend',
  noSkill: 'None',
  allSkills: 'Possible Skills',

  start: 'Start',
  pause: 'Pause',
  resume: 'Resume',
  restart: 'Restart',
  exit: 'Exit',

  readyMessage: 'Choose a mode and snake style, then press Start.',
  pausedMessage: 'Game paused. Press Resume or Space to continue.',
  overMessage: 'Game Over! Restart and try again.',
  gameOverTitle: 'Game Over',
  gameOverSubtitle: 'Challenge ended. Try again and beat your best score!',

  rewardTitle: 'Reward Wheel',
  rewardButton: 'Claim and Continue',
  skillEffect: 'Skill Effect',
  howToUse: 'How to Use',

  controls: 'Controls',
  space: 'Space: Pause / Resume',

  rules: 'Game Rules',
  ruleFood: 'Eat 🍎 food = +1 point. In Treasure mode, 💎 golden food = +3 points.',
  ruleHazard: 'Wall, body, 🧱 obstacle, 💣 bomb, or 💥 explosion costs 1 life.',
  ruleBomb: '💣 bombs explode randomly. The blast covers a 3×3 area around the bomb.',
  ruleLevel: 'Every 5 points takes you to the next level. Speed, obstacles, and bombs increase.',
  ruleReward: 'Every 10 points triggers the reward wheel. Skills activate automatically.',
  ruleMode: 'Different modes change the gameplay, not only the difficulty.',

  activeEffects: 'Effect Timer',
  noActiveEffect: 'No active effect',

  legendFood: 'Food',
  legendGoldFood: 'Golden Food',
  legendObstacle: 'Obstacle',
  legendBomb: 'Bomb',
  legendExplosion: 'Explosion',

  secondPerStep: 's / step',
  second: 's',
}

const text: Record<Language, typeof zhText> = {
  zhTw: zhText,
  en: enText,
  ja: enText,
  ko: enText,
  es: enText,
  fr: enText,
  de: enText,
  pt: enText,
  ru: enText,
  ar: enText,
  hi: enText,
  id: enText,
  vi: enText,
  th: enText,
}

const rewardTexts: Record<Language, Record<RewardKey, [string, string, string]>> = {
  zhTw: {
    life1: ['生命 +1', '生命 +1，之後撞到危險物時可以多撐一次。', '自動生效，地圖上方會增加一顆紅色愛心。'],
    life2: ['生命 +2', '生命 +2，後面關卡容錯率更高。', '自動生效，地圖上方會增加兩顆紅色愛心。'],
    shorten: ['蛇身 -3', '蛇的身體縮短 3 格，轉彎和閃避更容易。', '自動生效，身體會立刻變短。'],
    removeObstacle: ['移除 3 個障礙', '移除場上 3 個 🧱 障礙物。', '自動生效，部分障礙會立刻消失。'],
    clearBomb: ['清除所有炸彈', '清除場上所有 💣 炸彈。', '自動生效，所有炸彈會消失。'],
    slow: ['緩速 10 秒', '接下來 10 秒蛇的速度變慢。', '自動生效，趁這段時間吃食物或找安全路線。'],
    shield: ['護盾 10 秒', '10 秒內碰到障礙、炸彈或爆炸不會死亡。', '自動生效，蛇頭變成金色時可以安全通過危險物。'],
    hideObstacle: ['障礙失效 10 秒', '10 秒內 🧱 障礙暫時失效，可以安全通過。', '自動生效，趁障礙失效時快速移動。'],
  },
  en: {
    life1: ['Life +1', 'Gain 1 extra life for future mistakes.', 'Works automatically. One red heart is added.'],
    life2: ['Life +2', 'Gain 2 extra lives for harder levels.', 'Works automatically. Two red hearts are added.'],
    shorten: ['Snake -3', 'Shortens the snake by 3 blocks.', 'Works automatically and makes turning easier.'],
    removeObstacle: ['Remove 3 Obstacles', 'Removes 3 🧱 obstacles.', 'Works automatically. Some obstacles disappear.'],
    clearBomb: ['Clear All Bombs', 'Removes all 💣 bombs.', 'Works automatically. All bombs disappear.'],
    slow: ['Slow 10s', 'The snake slows down for 10 seconds.', 'Use this time to eat food or find a safe route.'],
    shield: ['Shield 10s', 'For 10 seconds, obstacles, bombs, and explosions do not hurt you.', 'Works automatically. Golden head means shield is active.'],
    hideObstacle: ['Disable Obstacles 10s', '🧱 obstacles are disabled for 10 seconds.', 'Move through obstacles safely while active.'],
  },
  ja: {} as Record<RewardKey, [string, string, string]>,
  ko: {} as Record<RewardKey, [string, string, string]>,
  es: {} as Record<RewardKey, [string, string, string]>,
  fr: {} as Record<RewardKey, [string, string, string]>,
  de: {} as Record<RewardKey, [string, string, string]>,
  pt: {} as Record<RewardKey, [string, string, string]>,
  ru: {} as Record<RewardKey, [string, string, string]>,
  ar: {} as Record<RewardKey, [string, string, string]>,
  hi: {} as Record<RewardKey, [string, string, string]>,
  id: {} as Record<RewardKey, [string, string, string]>,
  vi: {} as Record<RewardKey, [string, string, string]>,
  th: {} as Record<RewardKey, [string, string, string]>,
}

for (const lang of ['ja', 'ko', 'es', 'fr', 'de', 'pt', 'ru', 'ar', 'hi', 'id', 'vi', 'th'] as Language[]) {
  rewardTexts[lang] = rewardTexts.en
}

const t = computed(() => text[language.value])
const r = computed(() => rewardTexts[language.value])

function getText(key: keyof typeof zhText) {
  return t.value[key]
}

function getModeDesc(mode: GameMode) {
  return t.value[`${mode}Desc` as keyof typeof zhText]
}

const rewardKeys: RewardKey[] = [
  'life1',
  'life2',
  'shorten',
  'removeObstacle',
  'clearBomb',
  'slow',
  'shield',
  'hideObstacle',
]

const currentSkillName = computed(() => {
  if (!currentSkillKey.value) return t.value.noSkill
  return r.value[currentSkillKey.value][0]
})

const rewardName = computed(() => {
  if (!rewardKey.value) return ''
  return r.value[rewardKey.value][0]
})

const rewardDescription = computed(() => {
  if (!rewardKey.value) return ''
  return r.value[rewardKey.value][1]
})

const rewardHowToUse = computed(() => {
  if (!rewardKey.value) return ''
  return r.value[rewardKey.value][2]
})

const statusText = computed(() => {
  if (status.value === 'ready') return t.value.ready
  if (status.value === 'playing') return t.value.playing
  if (status.value === 'paused') return t.value.paused
  if (status.value === 'reward') return t.value.reward
  return t.value.over
})

const modeDescription = computed(() => getModeDesc(gameMode.value))
const nextRewardPoint = computed(() => Math.floor(score.value / 10) * 10 + 10)

const baseSpeed = computed(() => {
  if (gameMode.value === 'time') return 120
  if (gameMode.value === 'survival') return 125
  if (gameMode.value === 'portal') return 135
  return 140
})

const snakeSpeedSeconds = computed(() => {
  const realSpeed = activeSlow.value ? speed.value + 90 : speed.value
  return (realSpeed / 1000).toFixed(2)
})

function getLeftSecond(endTime: number) {
  const leftMs = endTime - now.value
  if (leftMs <= 0) return 0
  return Math.ceil(leftMs / 1000)
}

const shieldLeft = computed(() => getLeftSecond(shieldUntil.value))
const slowLeft = computed(() => getLeftSecond(slowUntil.value))
const hideObstacleLeft = computed(() => getLeftSecond(hideObstacleUntil.value))
const invincibleLeft = computed(() => getLeftSecond(invincibleUntil.value))
const goldenFoodLeft = computed(() => getLeftSecond(goldenFoodUntil.value))
const timeLeft = computed(() => getLeftSecond(timeAttackUntil.value))
const survivalPointLeft = computed(() => getLeftSecond(survivalNextPointAt.value))

const activeShield = computed(() => shieldLeft.value > 0)
const activeSlow = computed(() => slowLeft.value > 0)
const obstaclesHidden = computed(() => hideObstacleLeft.value > 0)
const isInvincible = computed(() => invincibleLeft.value > 0)
const activeGoldenFood = computed(() => goldenFood.value !== null && goldenFoodLeft.value > 0)

const cells = computed(() => {
  const result: Position[] = []

  for (let y = 0; y < boardSize; y++) {
    for (let x = 0; x < boardSize; x++) {
      result.push({ x, y })
    }
  }

  return result
})

function same(a: Position, b: Position) {
  return a.x === b.x && a.y === b.y
}

function inBoard(p: Position) {
  return p.x >= 0 && p.x < boardSize && p.y >= 0 && p.y < boardSize
}

function isSnakeHead(cell: Position) {
  return same(snake.value[0]!, cell)
}

function isSnakeBody(cell: Position) {
  return snake.value.slice(1).some((part) => same(part, cell))
}

function isFood(cell: Position) {
  return same(food.value, cell)
}

function isGoldenFood(cell: Position) {
  return activeGoldenFood.value && goldenFood.value !== null && same(goldenFood.value, cell)
}

function isObstacle(cell: Position) {
  return !obstaclesHidden.value && obstacles.value.some((item) => same(item, cell))
}

function isBomb(cell: Position) {
  return bombs.value.some((item) => same(item, cell))
}

function isExplosion(cell: Position) {
  return explosions.value.some((item) => same(item, cell) && item.endAt > now.value)
}

function getCellClass(cell: Position) {
  if (isSnakeHead(cell)) {
    if (activeShield.value) return 'cell snake-head shield'
    if (isInvincible.value) return 'cell snake-head invincible'
    return 'cell snake-head'
  }

  if (isSnakeBody(cell)) return 'cell snake-body'
  if (isExplosion(cell)) return 'cell explosion'
  if (isGoldenFood(cell)) return 'cell gold-food'
  if (isFood(cell)) return 'cell food'
  if (isBomb(cell)) return 'cell bomb'
  if (isObstacle(cell)) return 'cell obstacle'
  return 'cell'
}

function getSnakeEmoji(isHead: boolean) {
  if (snakeSkin.value === 'snake') return isHead ? '🐍' : '🟢'
  if (snakeSkin.value === 'dragon') return isHead ? '🐉' : '🟢'
  if (snakeSkin.value === 'robot') return isHead ? '🤖' : '⚙️'
  if (snakeSkin.value === 'cat') return isHead ? '😺' : '🐾'
  if (snakeSkin.value === 'avatar' && !avatarImage.value) return isHead ? '🙂' : '🟢'
  return ''
}

function shouldShowAvatarHead(cell: Position) {
  return isSnakeHead(cell) && snakeSkin.value === 'avatar' && avatarImage.value !== ''
}

function getCellContent(cell: Position) {
  if (isSnakeHead(cell)) return getSnakeEmoji(true)
  if (isSnakeBody(cell)) return getSnakeEmoji(false)
  if (isExplosion(cell)) return '💥'
  if (isGoldenFood(cell)) return '💎'
  if (isFood(cell)) return '🍎'
  if (isBomb(cell)) return '💣'
  if (isObstacle(cell)) return obstaclesHidden.value ? '' : '🧱'
  return ''
}

function isBlockedForPath(p: Position) {
  if (!inBoard(p)) return true
  if (!obstaclesHidden.value && obstacles.value.some((item) => same(item, p))) return true
  if (bombs.value.some((item) => same(item, p))) return true
  return false
}

function freeNeighborCount(p: Position) {
  const dirs = [
    { x: 0, y: -1 },
    { x: 0, y: 1 },
    { x: -1, y: 0 },
    { x: 1, y: 0 },
  ]

  return dirs.filter((d) => {
    const next = { x: p.x + d.x, y: p.y + d.y }
    return inBoard(next) && !isBlockedForPath(next)
  }).length
}

function pathExists(start: Position, target: Position) {
  const queue: Position[] = [start]
  const visited = new Set<string>()

  visited.add(`${start.x},${start.y}`)

  const dirs = [
    { x: 0, y: -1 },
    { x: 0, y: 1 },
    { x: -1, y: 0 },
    { x: 1, y: 0 },
  ]

  while (queue.length > 0) {
    const current = queue.shift()!

    if (same(current, target)) return true

    for (const d of dirs) {
      const next = { x: current.x + d.x, y: current.y + d.y }
      const key = `${next.x},${next.y}`

      if (visited.has(key)) continue
      if (!inBoard(next)) continue
      if (!same(next, target) && isBlockedForPath(next)) continue

      visited.add(key)
      queue.push(next)
    }
  }

  return false
}

function occupied(p: Position) {
  return (
    snake.value.some((part) => same(part, p)) ||
    same(food.value, p) ||
    (goldenFood.value !== null && same(goldenFood.value, p)) ||
    obstacles.value.some((item) => same(item, p)) ||
    bombs.value.some((item) => same(item, p))
  )
}

function farFromSnakeHead(p: Position) {
  const head = snake.value[0]!
  const distance = Math.abs(head.x - p.x) + Math.abs(head.y - p.y)
  return distance >= 3
}

function randomPosition() {
  return {
    x: Math.floor(Math.random() * boardSize),
    y: Math.floor(Math.random() * boardSize),
  }
}

function createFood() {
  for (let i = 0; i < 500; i++) {
    const candidate = randomPosition()

    if (occupied(candidate)) continue
    if (freeNeighborCount(candidate) < 1) continue

    const oldFood = food.value
    food.value = candidate

    if (pathExists(snake.value[0]!, candidate)) return

    food.value = oldFood
  }

  food.value = { x: 5, y: 5 }
}

function createGoldenFood() {
  if (gameMode.value !== 'treasure') return

  for (let i = 0; i < 400; i++) {
    const candidate = randomPosition()

    if (occupied(candidate)) continue
    if (freeNeighborCount(candidate) < 1) continue

    goldenFood.value = candidate
    goldenFoodUntil.value = Date.now() + 8000
    return
  }
}

function nextBombExplodeTime() {
  return Date.now() + 4000 + Math.floor(Math.random() * 6000)
}

function addSafeHazard(kind: 'obstacle' | 'bomb') {
  for (let i = 0; i < 400; i++) {
    const candidate = randomPosition()

    if (occupied(candidate)) continue
    if (!farFromSnakeHead(candidate)) continue
    if (Math.abs(candidate.x - food.value.x) + Math.abs(candidate.y - food.value.y) <= 1) continue

    if (kind === 'obstacle') {
      obstacles.value.push(candidate)
    } else {
      bombs.value.push({
        id: bombIdSeed++,
        x: candidate.x,
        y: candidate.y,
        explodeAt: nextBombExplodeTime(),
      })
    }

    const stillHasPath = pathExists(snake.value[0]!, food.value)
    const headHasSpace = freeNeighborCount(snake.value[0]!) >= 2
    const foodHasSpace = freeNeighborCount(food.value) >= 1

    if (stillHasPath && headHasSpace && foodHasSpace) return

    if (kind === 'obstacle') {
      obstacles.value.pop()
    } else {
      bombs.value.pop()
    }
  }
}

function getBlastArea(center: Position) {
  const area: Position[] = []

  for (let y = center.y - 1; y <= center.y + 1; y++) {
    for (let x = center.x - 1; x <= center.x + 1; x++) {
      const p = { x, y }

      if (inBoard(p)) {
        area.push(p)
      }
    }
  }

  return area
}

function triggerBombExplosion(bomb: Bomb) {
  const area = getBlastArea(bomb)

  bombs.value = bombs.value.filter((item) => item.id !== bomb.id)

  const endAt = Date.now() + explosionDurationMs

  for (const p of area) {
    explosions.value.push({
      id: explosionIdSeed++,
      x: p.x,
      y: p.y,
      endAt,
    })
  }

  const hitSnake = snake.value.some((part) => area.some((p) => same(p, part)))

  if (hitSnake && status.value === 'playing' && !activeShield.value && !isInvincible.value) {
    respawnAfterHit()
  }
}

function checkBombs() {
  if (status.value !== 'playing') return

  const dueBombs = bombs.value.filter((bomb) => bomb.explodeAt <= Date.now())

  for (const bomb of dueBombs) {
    triggerBombExplosion(bomb)
  }

  explosions.value = explosions.value.filter((item) => item.endAt > Date.now())
}

function createRespawnSnake(length: number) {
  const result: Position[] = [{ x: 10, y: 10 }]
  const maxLength = boardSize * boardSize
  const targetLength = Math.min(Math.max(1, length), maxLength)

  let y = 10
  let x = 9

  while (result.length < targetLength && x >= 0) {
    result.push({ x, y })
    x--
  }

  y = 11
  let goRight = true

  while (result.length < targetLength && y < boardSize) {
    if (goRight) {
      for (let col = 0; col < boardSize && result.length < targetLength; col++) {
        result.push({ x: col, y })
      }
    } else {
      for (let col = boardSize - 1; col >= 0 && result.length < targetLength; col--) {
        result.push({ x: col, y })
      }
    }

    goRight = !goRight
    y++
  }

  y = 9
  goRight = false

  while (result.length < targetLength && y >= 0) {
    if (goRight) {
      for (let col = 0; col < boardSize && result.length < targetLength; col++) {
        result.push({ x: col, y })
      }
    } else {
      for (let col = boardSize - 1; col >= 0 && result.length < targetLength; col--) {
        result.push({ x: col, y })
      }
    }

    goRight = !goRight
    y--
  }

  return result
}

function clearSpawnZone() {
  const spawn = { x: 10, y: 10 }

  obstacles.value = obstacles.value.filter((item) => {
    const distance = Math.abs(item.x - spawn.x) + Math.abs(item.y - spawn.y)
    return distance > 2
  })

  bombs.value = bombs.value.filter((item) => {
    const distance = Math.abs(item.x - spawn.x) + Math.abs(item.y - spawn.y)
    return distance > 2
  })

  explosions.value = explosions.value.filter((item) => {
    const distance = Math.abs(item.x - spawn.x) + Math.abs(item.y - spawn.y)
    return distance > 2
  })
}

function clearHazardsOnSnake() {
  obstacles.value = obstacles.value.filter((item) => !snake.value.some((part) => same(part, item)))
  bombs.value = bombs.value.filter((item) => !snake.value.some((part) => same(part, item)))
  explosions.value = explosions.value.filter((item) => !snake.value.some((part) => same(part, item)))
}

function clearGameTimer() {
  if (timer !== undefined) {
    clearInterval(timer)
    timer = undefined
  }
}

function startTimer() {
  clearGameTimer()

  let realSpeed = speed.value

  if (activeSlow.value) {
    realSpeed += 90
  }

  timer = window.setInterval(moveSnake, realSpeed)
}

function getInitialLives() {
  if (gameMode.value === 'time') return 2
  if (gameMode.value === 'portal') return 2
  if (gameMode.value === 'survival') return 2
  return 1
}

function resetGameData() {
  snake.value = [{ x: 10, y: 10 }]
  food.value = { x: 5, y: 5 }
  goldenFood.value = null
  obstacles.value = []
  bombs.value = []
  explosions.value = []

  direction.value = { x: 1, y: 0 }
  nextDirection.value = { x: 1, y: 0 }

  score.value = 0
  lives.value = getInitialLives()
  level.value = 1
  speed.value = baseSpeed.value

  lastDifficultyLevel.value = 0
  lastRewardScore.value = 0

  currentSkillKey.value = null
  rewardKey.value = null

  shieldUntil.value = 0
  slowUntil.value = 0
  hideObstacleUntil.value = 0
  invincibleUntil.value = 0
  goldenFoodUntil.value = 0
  timeAttackUntil.value = 0
  survivalNextPointAt.value = 0
  wasSlowActive = false

  if (gameMode.value === 'time') {
    timeAttackUntil.value = Date.now() + timeModeStartMs
  }

  if (gameMode.value === 'survival') {
    survivalNextPointAt.value = Date.now() + 4000
    addSafeHazard('obstacle')
    addSafeHazard('bomb')
  }

  createFood()
}

function startGame() {
  resetGameData()
  status.value = 'playing'
  startTimer()
}

function pauseGame() {
  if (status.value !== 'playing') return

  status.value = 'paused'
  clearGameTimer()
}

function resumeGame() {
  if (status.value !== 'paused' && status.value !== 'reward') return

  status.value = 'playing'
  startTimer()
}

function restartGame() {
  clearGameTimer()
  startGame()
}

function exitGame() {
  clearGameTimer()
  resetGameData()
  status.value = 'ready'
}

function endGame() {
  status.value = 'over'
  clearGameTimer()

  if (score.value > bestScore.value) {
    bestScore.value = score.value
  }
}

function respawnAfterHit() {
  if (isInvincible.value) return

  const keepLength = snake.value.length

  lives.value--

  if (lives.value <= 0) {
    endGame()
    return
  }

  snake.value = createRespawnSnake(keepLength)

  direction.value = { x: 1, y: 0 }
  nextDirection.value = { x: 1, y: 0 }

  clearSpawnZone()
  clearHazardsOnSnake()

  invincibleUntil.value = Date.now() + respawnProtectionMs

  if (snake.value.some((part) => same(part, food.value)) || !pathExists(snake.value[0]!, food.value)) {
    createFood()
  }
}

function removeHazardAt(target: Position) {
  obstacles.value = obstacles.value.filter((item) => !same(item, target))
  bombs.value = bombs.value.filter((item) => !same(item, target))
  explosions.value = explosions.value.filter((item) => !same(item, target))
}

function handleCollision(target: Position, type: 'wall' | 'self' | 'obstacle' | 'bomb' | 'explosion') {
  if (isInvincible.value) return false

  if ((type === 'obstacle' || type === 'bomb' || type === 'explosion') && activeShield.value) {
    removeHazardAt(target)
    return false
  }

  if (type === 'bomb') {
    const bomb = bombs.value.find((item) => same(item, target))

    if (bomb) {
      triggerBombExplosion(bomb)
      return true
    }
  }

  if (type === 'obstacle') {
    obstacles.value = obstacles.value.filter((item) => !same(item, target))
  }

  respawnAfterHit()
  return true
}

function increaseDifficulty() {
  const difficultyStep = Math.floor(score.value / 5)

  if (difficultyStep <= 0) return
  if (difficultyStep <= lastDifficultyLevel.value) return

  lastDifficultyLevel.value = difficultyStep
  level.value = difficultyStep + 1

  const speedDrop =
    gameMode.value === 'time'
      ? difficultyStep * 12
      : gameMode.value === 'survival'
        ? difficultyStep * 11
        : difficultyStep * 9

  speed.value = Math.max(58, baseSpeed.value - speedDrop)

  addSafeHazard('obstacle')

  if (gameMode.value !== 'portal' && difficultyStep >= 2) {
    addSafeHazard('obstacle')
  }

  if (score.value >= 10) {
    addSafeHazard('bomb')
  }

  if (gameMode.value === 'survival') {
    addSafeHazard('bomb')
  }

  startTimer()
}

const rewards: { key: RewardKey; apply: () => void }[] = [
  {
    key: 'life1',
    apply: () => {
      lives.value += 1
    },
  },
  {
    key: 'life2',
    apply: () => {
      lives.value += 2
    },
  },
  {
    key: 'shorten',
    apply: () => {
      if (snake.value.length > 4) {
        snake.value = snake.value.slice(0, Math.max(1, snake.value.length - 3))
      }
    },
  },
  {
    key: 'removeObstacle',
    apply: () => {
      obstacles.value = obstacles.value.slice(0, Math.max(0, obstacles.value.length - 3))
    },
  },
  {
    key: 'clearBomb',
    apply: () => {
      bombs.value = []
      explosions.value = []
    },
  },
  {
    key: 'slow',
    apply: () => {
      slowUntil.value = Date.now() + skillDurationMs
      wasSlowActive = true

      if (status.value === 'playing') {
        startTimer()
      }
    },
  },
  {
    key: 'shield',
    apply: () => {
      shieldUntil.value = Date.now() + skillDurationMs
    },
  },
  {
    key: 'hideObstacle',
    apply: () => {
      hideObstacleUntil.value = Date.now() + skillDurationMs
    },
  },
]

function triggerReward() {
  if (score.value < lastRewardScore.value + 10) return

  lastRewardScore.value += 10
  clearGameTimer()
  status.value = 'reward'

  const reward = rewards[Math.floor(Math.random() * rewards.length)]!
  reward.apply()

  currentSkillKey.value = reward.key
  rewardKey.value = reward.key

  if (!pathExists(snake.value[0]!, food.value)) {
    createFood()
  }
}

function checkProgress() {
  increaseDifficulty()
  triggerReward()

  if (gameMode.value === 'treasure' && score.value > 0 && score.value % 7 === 0 && !activeGoldenFood.value) {
    createGoldenFood()
  }
}

function claimReward() {
  status.value = 'playing'
  startTimer()
}

function afterEatingFood() {
  score.value++

  if (gameMode.value === 'time') {
    timeAttackUntil.value += 3000
  }

  createFood()
  checkProgress()
}

function afterEatingGoldenFood() {
  score.value += 3
  goldenFood.value = null
  goldenFoodUntil.value = 0
  checkProgress()
}

function wrapPosition(p: Position) {
  return {
    x: (p.x + boardSize) % boardSize,
    y: (p.y + boardSize) % boardSize,
  }
}

function moveSnake() {
  if (status.value !== 'playing') return

  direction.value = nextDirection.value

  const head = snake.value[0]!
  let newHead = {
    x: head.x + direction.value.x,
    y: head.y + direction.value.y,
  }

  if (!inBoard(newHead)) {
    if (gameMode.value === 'portal') {
      newHead = wrapPosition(newHead)
    } else {
      handleCollision(newHead, 'wall')
      return
    }
  }

  if (snake.value.some((part) => same(part, newHead))) {
    handleCollision(newHead, 'self')
    return
  }

  if (isExplosion(newHead)) {
    const dead = handleCollision(newHead, 'explosion')
    if (dead) return
  }

  if (!obstaclesHidden.value && obstacles.value.some((item) => same(item, newHead))) {
    const dead = handleCollision(newHead, 'obstacle')
    if (dead) return
  }

  if (bombs.value.some((item) => same(item, newHead))) {
    const dead = handleCollision(newHead, 'bomb')
    if (dead) return
  }

  snake.value.unshift(newHead)

  if (activeGoldenFood.value && goldenFood.value !== null && same(newHead, goldenFood.value)) {
    afterEatingGoldenFood()
  } else if (same(newHead, food.value)) {
    afterEatingFood()
  } else {
    snake.value.pop()
  }
}

function changeDirection(event: KeyboardEvent) {
  const key = event.key.toLowerCase()

  const gameKeys = [
    'arrowup',
    'arrowdown',
    'arrowleft',
    'arrowright',
    'w',
    'a',
    's',
    'd',
    ' ',
  ]

  if (gameKeys.includes(key)) {
    event.preventDefault()
  }

  if (key === ' ' && status.value === 'playing') {
    pauseGame()
    return
  }

  if (key === ' ' && status.value === 'paused') {
    resumeGame()
    return
  }

  if (status.value !== 'playing') return

  const goingUp = direction.value.y === -1
  const goingDown = direction.value.y === 1
  const goingLeft = direction.value.x === -1
  const goingRight = direction.value.x === 1

  if ((key === 'arrowup' || key === 'w') && !goingDown) {
    nextDirection.value = { x: 0, y: -1 }
  } else if ((key === 'arrowdown' || key === 's') && !goingUp) {
    nextDirection.value = { x: 0, y: 1 }
  } else if ((key === 'arrowleft' || key === 'a') && !goingRight) {
    nextDirection.value = { x: -1, y: 0 }
  } else if ((key === 'arrowright' || key === 'd') && !goingLeft) {
    nextDirection.value = { x: 1, y: 0 }
  }
}

function handleModeClock() {
  if (status.value !== 'playing') return

  checkBombs()

  if (gameMode.value === 'time' && timeLeft.value <= 0) {
    endGame()
    return
  }

  if (gameMode.value === 'survival' && survivalPointLeft.value <= 0) {
    score.value++
    survivalNextPointAt.value = Date.now() + 4000
    checkProgress()
  }

  if (goldenFood.value !== null && goldenFoodLeft.value <= 0) {
    goldenFood.value = null
    goldenFoodUntil.value = 0
  }

  if (wasSlowActive && !activeSlow.value) {
    wasSlowActive = false
    startTimer()
  }
}

function openAvatarMenu() {
  avatarMenuOpen.value = true
}

function closeAvatarMenu() {
  avatarMenuOpen.value = false
  closeCamera()
}

async function openCamera() {
  cameraMessage.value = ''

  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    cameraMessage.value = getText('cameraDenied')
    return
  }

  try {
    cameraStream = await navigator.mediaDevices.getUserMedia({
      video: {
        width: 320,
        height: 320,
        facingMode: 'user',
      },
      audio: false,
    })

    if (videoRef.value) {
      videoRef.value.srcObject = cameraStream
      await videoRef.value.play()
    }

    cameraActive.value = true
    cameraMessage.value = getText('cameraReady')
  } catch {
    cameraMessage.value = getText('cameraDenied')
    cameraActive.value = false
  }
}

function capturePhoto() {
  const video = videoRef.value
  const canvas = canvasRef.value

  if (!video || !canvas) return

  const size = 260
  canvas.width = size
  canvas.height = size

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  ctx.clearRect(0, 0, size, size)
  ctx.save()
  ctx.beginPath()
  ctx.arc(size / 2, size / 2, size / 2, 0, Math.PI * 2)
  ctx.clip()

  const videoWidth = video.videoWidth || size
  const videoHeight = video.videoHeight || size
  const side = Math.min(videoWidth, videoHeight)
  const sx = (videoWidth - side) / 2
  const sy = (videoHeight - side) / 2

  ctx.drawImage(video, sx, sy, side, side, 0, 0, size, size)
  ctx.restore()

  avatarImage.value = canvas.toDataURL('image/png')
  snakeSkin.value = 'avatar'
  cameraMessage.value = getText('cameraCaptured')
}

function closeCamera() {
  if (cameraStream) {
    for (const track of cameraStream.getTracks()) {
      track.stop()
    }
  }

  cameraStream = null
  cameraActive.value = false

  if (videoRef.value) {
    videoRef.value.srcObject = null
  }
}

function clearAvatar() {
  avatarImage.value = ''

  if (snakeSkin.value === 'avatar') {
    snakeSkin.value = 'neon'
  }
}

onMounted(() => {
  window.addEventListener('keydown', changeDirection, { passive: false })

  clockTimer = window.setInterval(() => {
    now.value = Date.now()
    handleModeClock()
  }, 200)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', changeDirection)
  clearGameTimer()
  closeCamera()

  if (clockTimer !== undefined) {
    clearInterval(clockTimer)
  }
})
</script>

<template>
  <div class="page">
    <div class="background-circle circle-one"></div>
    <div class="background-circle circle-two"></div>
    <div class="background-circle circle-three"></div>

    <section class="game-shell">
      <header class="header">
        <div>
          <p class="tag">{{ t.project }}</p>
          <h1>{{ t.title }}</h1>
          <p class="description">{{ t.description }}</p>
        </div>

        <div class="top-boxes">
          <div class="select-box">
            <span>{{ t.language }}</span>
            <select v-model="language">
              <option v-for="item in languageOptions" :key="item.value" :value="item.value">
                {{ item.label }}
              </option>
            </select>
          </div>

          <div class="select-box">
            <span>{{ t.snakeStyle }}</span>
            <select
              v-model="snakeSkin"
              :disabled="status === 'playing' || status === 'paused' || status === 'reward'"
            >
              <option v-for="item in snakeSkinOptions" :key="item.value" :value="item.value">
                {{ item.icon }} {{ t[item.value] }}
              </option>
            </select>
          </div>

          <div class="select-box">
            <span>{{ t.mode }}</span>
            <select
              v-model="gameMode"
              :disabled="status === 'playing' || status === 'paused' || status === 'reward'"
            >
              <option v-for="item in modeOptions" :key="item.value" :value="item.value">
                {{ item.icon }} {{ t[item.value] }}
              </option>
            </select>
          </div>

          <div class="status-box">
            <span>{{ t.status }}</span>
            <strong>{{ statusText }}</strong>
          </div>
        </div>
      </header>

      <div class="mode-showcase">
        <button
          v-for="item in modeOptions"
          :key="item.value"
          class="mode-card"
          :class="{ selected: gameMode === item.value }"
          :disabled="status === 'playing' || status === 'paused' || status === 'reward'"
          @click="gameMode = item.value"
        >
          <span class="mode-icon">{{ item.icon }}</span>
          <strong>{{ t[item.value] }}</strong>
          <small>{{ getModeDesc(item.value) }}</small>
        </button>
      </div>

      <main class="game-layout">
        <div class="left-area">
          <div class="board-frame">
            <div class="board-title">
              <span>{{ t.boardTitle }}</span>

              <div class="life-bar">
                <span class="life-label">{{ t.lives }}</span>
                <span v-for="heart in lives" :key="heart" class="heart">❤️</span>
              </div>

              <span v-if="activeShield" class="skill-badge">{{ r.shield[0] }} {{ shieldLeft }}{{ t.second }}</span>
              <span v-else-if="activeSlow" class="skill-badge">{{ r.slow[0] }} {{ slowLeft }}{{ t.second }}</span>
              <span v-else-if="obstaclesHidden" class="skill-badge">{{ r.hideObstacle[0] }} {{ hideObstacleLeft }}{{ t.second }}</span>
              <span v-else-if="activeGoldenFood" class="skill-badge">💎 {{ goldenFoodLeft }}{{ t.second }}</span>
              <span v-else class="live-dot"></span>
            </div>

            <div class="board">
              <div v-for="(cell, index) in cells" :key="index" :class="getCellClass(cell)">
                <img v-if="shouldShowAvatarHead(cell)" :src="avatarImage" class="avatar-head" alt="avatar" />
                <span v-else class="cell-content">{{ getCellContent(cell) }}</span>
              </div>
            </div>
          </div>

          <div class="bottom-dashboard">
            <div class="effect-panel">
              <h3>{{ t.activeEffects }}</h3>

              <div v-if="activeShield || activeSlow || obstaclesHidden || activeGoldenFood" class="effect-list">
                <div v-if="activeShield" class="effect-item shield-effect">
                  <strong>{{ r.shield[0] }}</strong>
                  <span>{{ shieldLeft }}{{ t.second }}</span>
                </div>

                <div v-if="activeSlow" class="effect-item slow-effect">
                  <strong>{{ r.slow[0] }}</strong>
                  <span>{{ slowLeft }}{{ t.second }}</span>
                </div>

                <div v-if="obstaclesHidden" class="effect-item hide-effect">
                  <strong>{{ r.hideObstacle[0] }}</strong>
                  <span>{{ hideObstacleLeft }}{{ t.second }}</span>
                </div>

                <div v-if="activeGoldenFood" class="effect-item gold-effect">
                  <strong>💎 {{ t.legendGoldFood }}</strong>
                  <span>{{ goldenFoodLeft }}{{ t.second }}</span>
                </div>
              </div>

              <p v-else class="no-effect">{{ t.noActiveEffect }}</p>
            </div>

            <div class="legend-panel">
              <h3>{{ t.legend }}</h3>

              <div class="legend-list">
                <div class="legend-item"><span class="legend-emoji">🍎</span>{{ t.legendFood }}</div>
                <div class="legend-item"><span class="legend-emoji">💎</span>{{ t.legendGoldFood }}</div>
                <div class="legend-item"><span class="legend-emoji">🧱</span>{{ t.legendObstacle }}</div>
                <div class="legend-item"><span class="legend-emoji">💣</span>{{ t.legendBomb }}</div>
                <div class="legend-item"><span class="legend-emoji">💥</span>{{ t.legendExplosion }}</div>
              </div>
            </div>

            <div class="rules-panel">
              <h3>{{ t.rules }}</h3>
              <p>{{ t.ruleFood }}</p>
              <p>{{ t.ruleHazard }}</p>
              <p>{{ t.ruleBomb }}</p>
              <p>{{ t.ruleLevel }}</p>
              <p>{{ t.ruleReward }}</p>
              <p>{{ t.ruleMode }}</p>

              <div class="skill-list-title">{{ t.allSkills }}</div>
              <div class="skill-list">
                <span v-for="key in rewardKeys" :key="key" class="skill-pill">
                  {{ r[key][0] }}
                </span>
              </div>
            </div>
          </div>
        </div>

        <aside class="panel">
          <div class="score-grid">
            <div class="score-card">
              <p>{{ t.score }}</p>
              <h2>{{ score }}</h2>
            </div>

            <div class="score-card best">
              <p>{{ t.bestScore }}</p>
              <h2>{{ bestScore }}</h2>
            </div>

            <div class="mini-card">
              <p>{{ t.level }}</p>
              <strong>{{ level }}</strong>
            </div>

            <div class="mini-card skill-card">
              <p>{{ t.nextReward }}</p>
              <strong>{{ nextRewardPoint }}</strong>
            </div>
          </div>

          <div class="mode-info-card">
            <p>{{ t.modeFeature }}</p>
            <strong>{{ t[gameMode] }}</strong>
            <span>{{ modeDescription }}</span>
          </div>

          <button class="avatar-menu-button" @click="openAvatarMenu">
            <span class="avatar-menu-icon">
              <img v-if="avatarImage" :src="avatarImage" alt="avatar" />
              <span v-else>📸</span>
            </span>
            <span>
              <strong>{{ t.avatarMenu }}</strong>
              <small>{{ avatarImage ? t.avatar : t.cameraTitle }}</small>
            </span>
          </button>

          <div class="current-skill-box">
            <p>{{ t.skill }}</p>
            <strong>{{ currentSkillName }}</strong>
          </div>

          <div class="info-box">
            <p>{{ t.mode }}：{{ t[gameMode] }}</p>
            <p>{{ t.snakeStyle }}：{{ t[snakeSkin] }}</p>
            <p>{{ t.speed }}：{{ snakeSpeedSeconds }} {{ t.secondPerStep }}</p>
            <p>{{ t.obstacles }}：{{ obstacles.length }}</p>
            <p>{{ t.bombs }}：{{ bombs.length }}</p>
            <p v-if="gameMode === 'time'">{{ t.timeLeft }}：{{ timeLeft }}{{ t.second }}</p>
            <p v-if="gameMode === 'survival'">{{ t.nextSurvivalPoint }}：{{ survivalPointLeft }}{{ t.second }}</p>
          </div>

          <div class="button-group">
            <button v-if="status === 'ready' || status === 'over'" class="btn start" @click="startGame">
              {{ t.start }}
            </button>

            <button v-if="status === 'playing'" class="btn pause" @click="pauseGame">
              {{ t.pause }}
            </button>

            <button v-if="status === 'paused'" class="btn resume" @click="resumeGame">
              {{ t.resume }}
            </button>

            <button class="btn restart" @click="restartGame">
              {{ t.restart }}
            </button>

            <button class="btn exit" @click="exitGame">
              {{ t.exit }}
            </button>
          </div>

          <div v-if="status === 'ready'" class="message ready">
            {{ t.readyMessage }}
          </div>

          <div v-if="status === 'paused'" class="message paused">
            {{ t.pausedMessage }}
          </div>

          <div v-if="status === 'over'" class="message over">
            {{ t.overMessage }}
          </div>

          <div class="control-box">
            <h3>{{ t.controls }}</h3>
            <div class="keys">
              <span>↑</span>
              <span>↓</span>
              <span>←</span>
              <span>→</span>
            </div>
            <div class="keys">
              <span>W</span>
              <span>A</span>
              <span>S</span>
              <span>D</span>
            </div>
            <p>{{ t.space }}</p>
          </div>
        </aside>
      </main>
    </section>

    <div v-if="avatarMenuOpen" class="avatar-overlay">
      <div class="avatar-modal">
        <div class="avatar-modal-header">
          <div>
            <p>{{ t.avatarMenu }}</p>
            <h2>{{ t.cameraTitle }}</h2>
          </div>

          <button class="avatar-close-x" @click="closeAvatarMenu">×</button>
        </div>

        <div class="avatar-modal-body">
          <div class="camera-large-preview">
            <video
              ref="videoRef"
              class="camera-video"
              autoplay
              playsinline
              muted
              :class="{ hidden: !cameraActive }"
            ></video>

            <div v-if="!cameraActive && !avatarImage" class="camera-placeholder-large">
              📸
            </div>

            <img
              v-if="avatarImage && !cameraActive"
              :src="avatarImage"
              class="avatar-preview-large"
              alt="avatar preview"
            />

            <canvas ref="canvasRef" class="hidden-canvas"></canvas>
          </div>

          <div class="avatar-modal-info">
            <p>{{ t.cameraHint }}</p>
            <p v-if="cameraMessage" class="camera-message">{{ cameraMessage }}</p>

            <div class="avatar-action-grid">
              <button class="mini-btn camera-open" @click="openCamera">
                {{ t.openCamera }}
              </button>

              <button class="mini-btn camera-capture" :disabled="!cameraActive" @click="capturePhoto">
                {{ t.capturePhoto }}
              </button>

              <button class="mini-btn camera-close" @click="closeCamera">
                {{ t.closeCamera }}
              </button>

              <button class="mini-btn camera-clear" @click="clearAvatar">
                {{ t.clearAvatar }}
              </button>
            </div>

            <button class="btn avatar-done" @click="closeAvatarMenu">
              {{ t.closeMenu }}
            </button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="status === 'reward'" class="reward-overlay">
      <div class="reward-card big-reward-card">
        <div class="wheel">
          <div class="wheel-center">🎁</div>
        </div>

        <p class="reward-small-title">{{ t.rewardTitle }}</p>

        <h2 class="reward-main-title">
          {{ rewardName }}
        </h2>

        <div class="reward-detail-box">
          <h3>{{ t.skillEffect }}</h3>
          <p>{{ rewardDescription }}</p>
        </div>

        <div class="reward-detail-box use-box">
          <h3>{{ t.howToUse }}</h3>
          <p>{{ rewardHowToUse }}</p>
        </div>

        <button class="btn resume reward-btn" @click="claimReward">
          {{ t.rewardButton }}
        </button>
      </div>
    </div>

    <div v-if="status === 'over'" class="game-over-overlay">
      <div class="game-over-card">
        <div class="game-over-icon">💥</div>
        <p class="game-over-small">{{ t.status }}</p>
        <h2>{{ t.gameOverTitle }}</h2>
        <p class="game-over-subtitle">{{ t.gameOverSubtitle }}</p>

        <div class="game-over-score-box">
          <div>
            <span>{{ t.finalScore }}</span>
            <strong>{{ score }}</strong>
          </div>

          <div>
            <span>{{ t.bestScore }}</span>
            <strong>{{ bestScore }}</strong>
          </div>
        </div>

        <div class="game-over-buttons">
          <button class="btn start game-over-btn" @click="restartGame">
            {{ t.restart }}
          </button>

          <button class="btn exit game-over-btn" @click="exitGame">
            {{ t.exit }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

.page {
  min-height: 100vh;
  position: relative;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 16px;
  color: white;
  font-family:
    Arial,
    "Microsoft JhengHei",
    sans-serif;
  background:
    radial-gradient(circle at 10% 10%, rgba(236, 72, 153, 0.55), transparent 28%),
    radial-gradient(circle at 90% 80%, rgba(34, 211, 238, 0.5), transparent 28%),
    radial-gradient(circle at 50% 20%, rgba(250, 204, 21, 0.18), transparent 20%),
    linear-gradient(135deg, #020617, #111827, #1e1b4b, #0f172a);
}

.background-circle {
  position: absolute;
  border-radius: 50%;
  filter: blur(5px);
  opacity: 0.42;
  animation: float 5s infinite alternate ease-in-out;
}

.circle-one {
  width: 150px;
  height: 150px;
  left: 7%;
  top: 13%;
  background: #f97316;
}

.circle-two {
  width: 220px;
  height: 220px;
  right: 8%;
  bottom: 8%;
  background: #22c55e;
}

.circle-three {
  width: 120px;
  height: 120px;
  right: 22%;
  top: 11%;
  background: #ec4899;
}

@keyframes float {
  from {
    transform: translateY(0);
  }

  to {
    transform: translateY(-25px);
  }
}

.game-shell {
  position: relative;
  z-index: 1;
  width: 1320px;
  padding: 18px;
  border: 1px solid rgba(255, 255, 255, 0.18);
  border-radius: 28px;
  background:
    linear-gradient(135deg, rgba(15, 23, 42, 0.88), rgba(30, 41, 59, 0.72)),
    rgba(15, 23, 42, 0.78);
  box-shadow: 0 30px 80px rgba(0, 0, 0, 0.48);
  backdrop-filter: blur(18px);
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 22px;
  margin-bottom: 14px;
}

.tag {
  display: inline-block;
  margin: 0 0 6px;
  padding: 5px 11px;
  color: #67e8f9;
  background: rgba(8, 145, 178, 0.22);
  border: 1px solid rgba(103, 232, 249, 0.4);
  border-radius: 999px;
  font-size: 13px;
  letter-spacing: 1px;
}

h1 {
  margin: 0;
  font-size: 40px;
  letter-spacing: 1px;
  background: linear-gradient(90deg, #facc15, #fb7185, #38bdf8, #4ade80);
  -webkit-background-clip: text;
  color: transparent;
}

.description {
  margin: 6px 0 0;
  color: #cbd5e1;
  font-size: 15px;
}

.top-boxes {
  display: flex;
  gap: 10px;
  align-items: stretch;
}

.select-box {
  width: 145px;
  padding: 11px;
  border-radius: 18px;
  text-align: center;
  background: linear-gradient(135deg, rgba(124, 45, 18, 0.95), rgba(190, 24, 93, 0.95));
  box-shadow: 0 12px 30px rgba(244, 114, 182, 0.16);
}

.select-box span,
.status-box span {
  display: block;
  color: #ffe4e6;
  font-size: 12px;
  margin-bottom: 7px;
}

.select-box select {
  width: 100%;
  padding: 8px 9px;
  border: none;
  outline: none;
  border-radius: 12px;
  color: #0f172a;
  background: #fff7ed;
  font-size: 13px;
  font-weight: bold;
  cursor: pointer;
}

.select-box select:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.status-box {
  width: 118px;
  padding: 11px;
  border-radius: 18px;
  text-align: center;
  background: linear-gradient(135deg, #312e81, #0f766e);
  box-shadow: 0 12px 30px rgba(34, 211, 238, 0.18);
}

.status-box strong {
  font-size: 19px;
}

.mode-showcase {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 10px;
  margin-bottom: 14px;
}

.mode-card {
  min-height: 92px;
  padding: 12px;
  border: 1px solid rgba(148, 163, 184, 0.24);
  border-radius: 20px;
  color: white;
  text-align: left;
  background:
    linear-gradient(135deg, rgba(30, 41, 59, 0.72), rgba(15, 23, 42, 0.9));
  cursor: pointer;
  transition:
    transform 0.15s,
    border 0.15s,
    box-shadow 0.15s;
}

.mode-card:hover {
  transform: translateY(-3px);
  border-color: rgba(56, 189, 248, 0.7);
  box-shadow: 0 15px 28px rgba(56, 189, 248, 0.16);
}

.mode-card:disabled {
  cursor: not-allowed;
  opacity: 0.7;
}

.mode-card.selected {
  border-color: rgba(250, 204, 21, 0.85);
  box-shadow: 0 0 26px rgba(250, 204, 21, 0.22);
  background:
    linear-gradient(135deg, rgba(88, 28, 135, 0.88), rgba(190, 24, 93, 0.78));
}

.mode-icon {
  display: block;
  font-size: 24px;
  margin-bottom: 5px;
}

.mode-card strong {
  display: block;
  margin-bottom: 4px;
  font-size: 15px;
}

.mode-card small {
  display: block;
  color: #cbd5e1;
  font-size: 12px;
  line-height: 1.35;
}

.game-layout {
  display: flex;
  gap: 20px;
  align-items: flex-start;
}

.left-area {
  flex: 1;
  min-width: 0;
}

.board-frame {
  padding: 14px;
  border-radius: 24px;
  background:
    radial-gradient(circle at top, rgba(56, 189, 248, 0.16), transparent 35%),
    linear-gradient(135deg, rgba(30, 41, 59, 0.92), rgba(15, 23, 42, 0.92));
  border: 1px solid rgba(148, 163, 184, 0.25);
}

.board-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  margin-bottom: 11px;
  color: #e0f2fe;
  font-weight: bold;
}

.life-bar {
  display: flex;
  align-items: center;
  gap: 6px;
  min-height: 38px;
  padding: 7px 12px;
  border-radius: 999px;
  background: rgba(127, 29, 29, 0.45);
  border: 1px solid rgba(248, 113, 113, 0.5);
  box-shadow: 0 0 18px rgba(239, 68, 68, 0.25);
}

.life-label {
  color: #fecaca;
  font-weight: bold;
  margin-right: 4px;
}

.heart {
  font-size: 21px;
  filter: drop-shadow(0 0 6px rgba(248, 113, 113, 0.8));
  animation: heartBeat 0.9s infinite alternate;
}

@keyframes heartBeat {
  from {
    transform: scale(1);
  }

  to {
    transform: scale(1.16);
  }
}

.live-dot {
  width: 13px;
  height: 13px;
  border-radius: 50%;
  background: #22c55e;
  box-shadow: 0 0 15px #22c55e;
}

.skill-badge {
  padding: 6px 12px;
  border-radius: 999px;
  color: #0f172a;
  background: linear-gradient(135deg, #fef08a, #38bdf8);
  font-size: 13px;
  box-shadow: 0 0 18px rgba(250, 204, 21, 0.55);
}

.board {
  width: 560px;
  height: 560px;
  display: grid;
  grid-template-columns: repeat(20, 1fr);
  grid-template-rows: repeat(20, 1fr);
  border: 5px solid #38bdf8;
  border-radius: 18px;
  overflow: hidden;
  margin: 0 auto;
  background:
    linear-gradient(45deg, rgba(14, 165, 233, 0.12) 25%, transparent 25%),
    linear-gradient(-45deg, rgba(168, 85, 247, 0.12) 25%, transparent 25%),
    #020617;
  box-shadow:
    inset 0 0 30px rgba(56, 189, 248, 0.3),
    0 0 35px rgba(56, 189, 248, 0.25);
}

.cell {
  position: relative;
  display: grid;
  place-items: center;
  border: 1px solid rgba(51, 65, 85, 0.7);
  background: rgba(15, 23, 42, 0.72);
  font-size: 20px;
  line-height: 1;
}

.cell-content {
  position: relative;
  z-index: 2;
  filter: drop-shadow(0 0 5px rgba(255, 255, 255, 0.45));
}

.avatar-head {
  width: 92%;
  height: 92%;
  object-fit: cover;
  border-radius: 50%;
  border: 2px solid #fef08a;
  box-shadow: 0 0 12px #facc15;
}

.snake-head {
  border-radius: 9px;
  background: radial-gradient(circle, #fef08a, #22c55e 55%, #15803d);
  box-shadow: 0 0 14px #bef264;
}

.snake-head.shield {
  background: radial-gradient(circle, #ffffff, #facc15 45%, #22c55e);
  box-shadow: 0 0 24px #facc15;
}

.snake-head.invincible {
  background: radial-gradient(circle, #ffffff, #93c5fd 45%, #2563eb);
  box-shadow: 0 0 24px #60a5fa;
  animation: invincibleBlink 0.25s infinite alternate;
}

.snake-body {
  border-radius: 7px;
  background: linear-gradient(135deg, #86efac, #16a34a);
  box-shadow: inset 0 0 8px rgba(255, 255, 255, 0.4);
}

.food {
  border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, #ffffff, #fb7185 35%, #e11d48 70%);
  box-shadow: 0 0 18px #fb7185;
  animation: foodPulse 0.75s infinite alternate;
}

.gold-food {
  border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, #ffffff, #facc15 35%, #ca8a04 70%);
  box-shadow: 0 0 22px #facc15;
  animation: goldPulse 0.75s infinite alternate;
}

.obstacle {
  border-radius: 8px;
  background:
    linear-gradient(135deg, #c084fc, #7e22ce 45%, #3b0764),
    #6d28d9;
  box-shadow:
    inset 0 0 8px rgba(255, 255, 255, 0.35),
    0 0 10px rgba(168, 85, 247, 0.45);
}

.bomb {
  border-radius: 50%;
  background:
    radial-gradient(circle at 35% 28%, #ffffff 0 8%, #67e8f9 15%, #0284c7 35%, #1e3a8a 62%, #020617 85%);
  border: 2px solid #bae6fd;
  box-shadow:
    0 0 20px #38bdf8,
    0 0 8px #ffffff inset;
  animation: bombBlink 0.55s infinite alternate;
}

.explosion {
  border-radius: 6px;
  background:
    radial-gradient(circle, #ffffff 0 8%, #facc15 20%, #f97316 50%, #dc2626 80%);
  box-shadow:
    0 0 22px #f97316,
    0 0 18px #ef4444 inset;
  animation: explosionPulse 0.28s infinite alternate;
}

@keyframes foodPulse {
  from {
    transform: scale(0.78);
  }

  to {
    transform: scale(1.05);
  }
}

@keyframes goldPulse {
  from {
    transform: scale(0.8) rotate(0deg);
  }

  to {
    transform: scale(1.1) rotate(8deg);
  }
}

@keyframes bombBlink {
  from {
    transform: scale(0.78);
    filter: brightness(0.85);
  }

  to {
    transform: scale(1.08);
    filter: brightness(1.45);
  }
}

@keyframes explosionPulse {
  from {
    transform: scale(0.9);
    filter: brightness(1);
  }

  to {
    transform: scale(1.12);
    filter: brightness(1.5);
  }
}

@keyframes invincibleBlink {
  from {
    opacity: 0.7;
  }

  to {
    opacity: 1;
  }
}

.bottom-dashboard {
  display: grid;
  grid-template-columns: 1.1fr 0.8fr 2fr;
  gap: 12px;
  margin-top: 12px;
}

.effect-panel,
.legend-panel,
.rules-panel {
  min-height: 132px;
  padding: 14px;
  border-radius: 18px;
  background: rgba(2, 6, 23, 0.58);
  border: 1px solid rgba(148, 163, 184, 0.22);
}

.effect-panel h3,
.legend-panel h3,
.rules-panel h3 {
  margin: 0 0 10px;
  color: #a5f3fc;
  font-size: 16px;
}

.effect-list {
  display: grid;
  gap: 8px;
}

.effect-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 10px;
  border-radius: 12px;
  font-size: 14px;
}

.effect-item span {
  font-size: 18px;
  font-weight: bold;
}

.shield-effect {
  background: rgba(250, 204, 21, 0.22);
  color: #fef9c3;
}

.slow-effect {
  background: rgba(56, 189, 248, 0.22);
  color: #e0f2fe;
}

.hide-effect {
  background: rgba(34, 197, 94, 0.22);
  color: #dcfce7;
}

.gold-effect {
  background: rgba(250, 204, 21, 0.25);
  color: #fef9c3;
}

.no-effect {
  margin: 0;
  color: #cbd5e1;
  font-size: 14px;
}

.legend-list {
  display: grid;
  gap: 10px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 9px;
  color: #e5e7eb;
  font-size: 14px;
}

.legend-emoji {
  width: 28px;
  height: 28px;
  display: grid;
  place-items: center;
  border-radius: 10px;
  background: rgba(15, 23, 42, 0.8);
  font-size: 20px;
}

.rules-panel p {
  margin: 4px 0;
  color: #dbeafe;
  font-size: 13px;
  line-height: 1.38;
}

.skill-list-title {
  margin: 8px 0 6px;
  color: #fde68a;
  font-size: 13px;
  font-weight: bold;
}

.skill-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.skill-pill {
  display: inline-block;
  padding: 4px 8px;
  border-radius: 999px;
  color: #f8fafc;
  background: rgba(124, 58, 237, 0.38);
  border: 1px solid rgba(196, 181, 253, 0.25);
  font-size: 12px;
}

.panel {
  width: 345px;
  padding: 18px;
  border-radius: 24px;
  background: rgba(2, 6, 23, 0.65);
  border: 1px solid rgba(148, 163, 184, 0.25);
}

.score-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.score-card,
.mini-card {
  padding: 14px;
  border-radius: 18px;
  background: linear-gradient(135deg, #4c1d95, #1d4ed8);
}

.score-card.best {
  background: linear-gradient(135deg, #b45309, #be123c);
}

.mini-card {
  background: linear-gradient(135deg, #0f766e, #0891b2);
}

.skill-card {
  background: linear-gradient(135deg, #7c2d12, #c2410c);
}

.score-card p,
.mini-card p {
  margin: 0;
  color: #e0e7ff;
  font-size: 13px;
}

.score-card h2 {
  margin: 8px 0 0;
  font-size: 38px;
}

.mini-card strong {
  display: block;
  margin-top: 8px;
  font-size: 28px;
}

.mode-info-card {
  margin-top: 12px;
  padding: 14px;
  border-radius: 18px;
  background:
    radial-gradient(circle at top right, rgba(250, 204, 21, 0.22), transparent 40%),
    linear-gradient(135deg, rgba(88, 28, 135, 0.9), rgba(15, 23, 42, 0.85));
  border: 1px solid rgba(250, 204, 21, 0.25);
}

.mode-info-card p {
  margin: 0 0 7px;
  color: #fde68a;
  font-size: 13px;
}

.mode-info-card strong {
  display: block;
  margin-bottom: 6px;
  font-size: 20px;
}

.mode-info-card span {
  color: #dbeafe;
  font-size: 13px;
  line-height: 1.4;
}

.avatar-menu-button {
  width: 100%;
  margin-top: 12px;
  padding: 12px;
  display: flex;
  align-items: center;
  gap: 12px;
  border: 1px solid rgba(125, 211, 252, 0.28);
  border-radius: 18px;
  color: white;
  text-align: left;
  cursor: pointer;
  background:
    radial-gradient(circle at top left, rgba(56, 189, 248, 0.26), transparent 45%),
    rgba(30, 41, 59, 0.75);
  transition:
    transform 0.15s,
    box-shadow 0.15s;
}

.avatar-menu-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 12px 25px rgba(56, 189, 248, 0.18);
}

.avatar-menu-icon {
  width: 48px;
  height: 48px;
  display: grid;
  place-items: center;
  overflow: hidden;
  border-radius: 50%;
  background: rgba(15, 23, 42, 0.8);
  border: 2px solid rgba(250, 204, 21, 0.7);
  font-size: 24px;
}

.avatar-menu-icon img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.avatar-menu-button strong {
  display: block;
  font-size: 15px;
}

.avatar-menu-button small {
  display: block;
  margin-top: 3px;
  color: #cbd5e1;
  font-size: 12px;
}

.current-skill-box {
  margin-top: 12px;
  padding: 14px;
  border-radius: 18px;
  background: linear-gradient(135deg, #581c87, #be185d);
  box-shadow: 0 0 20px rgba(236, 72, 153, 0.18);
}

.current-skill-box p {
  margin: 0 0 8px;
  color: #fce7f3;
  font-size: 13px;
}

.current-skill-box strong {
  font-size: 20px;
  color: white;
}

.info-box {
  padding: 12px;
  margin-top: 12px;
  border-radius: 16px;
  background: rgba(30, 41, 59, 0.72);
  color: #dbeafe;
  font-size: 13px;
}

.info-box p {
  margin: 5px 0;
}

.button-group {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  margin: 14px 0;
}

.btn {
  border: none;
  padding: 12px 10px;
  border-radius: 14px;
  color: white;
  font-size: 15px;
  font-weight: bold;
  cursor: pointer;
  transition:
    transform 0.15s,
    filter 0.15s;
}

.btn:hover {
  transform: translateY(-2px);
  filter: brightness(1.1);
}

.start {
  background: linear-gradient(135deg, #22c55e, #14b8a6);
}

.pause {
  background: linear-gradient(135deg, #f59e0b, #f97316);
}

.resume {
  background: linear-gradient(135deg, #06b6d4, #2563eb);
}

.restart {
  background: linear-gradient(135deg, #8b5cf6, #6366f1);
}

.exit {
  background: linear-gradient(135deg, #ef4444, #be123c);
}

.message {
  padding: 12px;
  margin-bottom: 12px;
  border-radius: 16px;
  font-weight: bold;
  font-size: 14px;
}

.ready {
  background: rgba(59, 130, 246, 0.22);
  color: #bfdbfe;
}

.paused {
  background: rgba(245, 158, 11, 0.22);
  color: #fde68a;
}

.over {
  background: rgba(239, 68, 68, 0.22);
  color: #fecaca;
}

.control-box {
  padding: 14px;
  margin-top: 12px;
  border-radius: 18px;
  background: rgba(30, 41, 59, 0.72);
  border: 1px solid rgba(148, 163, 184, 0.22);
}

.control-box h3 {
  margin: 0 0 10px;
  color: #a5f3fc;
}

.keys {
  display: flex;
  gap: 8px;
  margin-bottom: 9px;
}

.keys span {
  width: 34px;
  height: 32px;
  display: grid;
  place-items: center;
  border-radius: 10px;
  color: #0f172a;
  background: #e0f2fe;
  font-weight: bold;
}

.control-box p {
  margin: 8px 0 0;
  color: #cbd5e1;
  font-size: 13px;
}

.reward-overlay,
.game-over-overlay,
.avatar-overlay {
  position: fixed;
  inset: 0;
  z-index: 20;
  display: grid;
  place-items: center;
  background: rgba(2, 6, 23, 0.72);
  backdrop-filter: blur(8px);
}

.avatar-overlay {
  z-index: 25;
}

.avatar-modal {
  width: 720px;
  max-width: 92vw;
  padding: 24px;
  border-radius: 30px;
  background:
    radial-gradient(circle at top left, rgba(56, 189, 248, 0.28), transparent 42%),
    linear-gradient(135deg, rgba(15, 23, 42, 0.97), rgba(49, 46, 129, 0.95));
  border: 1px solid rgba(255, 255, 255, 0.22);
  box-shadow: 0 30px 90px rgba(0, 0, 0, 0.62);
}

.avatar-modal-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 20px;
  margin-bottom: 18px;
}

.avatar-modal-header p {
  margin: 0 0 6px;
  color: #fde68a;
  font-weight: bold;
  font-size: 14px;
}

.avatar-modal-header h2 {
  margin: 0;
  font-size: 34px;
  background: linear-gradient(90deg, #facc15, #38bdf8, #fb7185);
  -webkit-background-clip: text;
  color: transparent;
}

.avatar-close-x {
  width: 40px;
  height: 40px;
  border: none;
  border-radius: 50%;
  color: white;
  background: rgba(239, 68, 68, 0.82);
  font-size: 28px;
  cursor: pointer;
}

.avatar-modal-body {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 18px;
}

.camera-large-preview {
  height: 330px;
  display: grid;
  place-items: center;
  overflow: hidden;
  border-radius: 26px;
  background: rgba(15, 23, 42, 0.88);
  border: 1px solid rgba(148, 163, 184, 0.28);
}

.camera-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.camera-placeholder-large {
  font-size: 80px;
  opacity: 0.8;
}

.avatar-preview-large {
  width: 235px;
  height: 235px;
  object-fit: cover;
  border-radius: 50%;
  border: 5px solid #facc15;
  box-shadow: 0 0 30px rgba(250, 204, 21, 0.65);
}

.avatar-modal-info {
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.avatar-modal-info p {
  margin: 0 0 12px;
  color: #dbeafe;
  line-height: 1.5;
}

.camera-message {
  color: #fde68a !important;
  font-weight: bold;
}

.avatar-action-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.mini-btn {
  border: none;
  padding: 11px 8px;
  border-radius: 13px;
  color: white;
  font-size: 13px;
  font-weight: bold;
  cursor: pointer;
}

.mini-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.camera-open {
  background: linear-gradient(135deg, #06b6d4, #2563eb);
}

.camera-capture {
  background: linear-gradient(135deg, #22c55e, #16a34a);
}

.camera-close {
  background: linear-gradient(135deg, #f97316, #c2410c);
}

.camera-clear {
  background: linear-gradient(135deg, #ef4444, #be123c);
}

.avatar-done {
  width: 100%;
  margin-top: 14px;
  background: linear-gradient(135deg, #8b5cf6, #ec4899);
}

.hidden,
.hidden-canvas {
  display: none;
}

.reward-card,
.game-over-card {
  width: 420px;
  padding: 32px;
  border-radius: 28px;
  text-align: center;
  background: linear-gradient(135deg, #312e81, #be185d, #f59e0b);
  box-shadow: 0 30px 80px rgba(0, 0, 0, 0.55);
  border: 1px solid rgba(255, 255, 255, 0.3);
}

.big-reward-card {
  width: 560px;
  padding: 38px;
}

.wheel {
  width: 170px;
  height: 170px;
  margin: 0 auto 18px;
  border-radius: 50%;
  display: grid;
  place-items: center;
  background:
    conic-gradient(
      #facc15,
      #fb7185,
      #38bdf8,
      #4ade80,
      #a78bfa,
      #facc15
    );
  animation: spinWheel 1.2s ease-out;
  box-shadow: 0 0 35px rgba(250, 204, 21, 0.55);
}

.wheel-center {
  width: 76px;
  height: 76px;
  display: grid;
  place-items: center;
  border-radius: 50%;
  background: #0f172a;
  font-size: 34px;
}

@keyframes spinWheel {
  from {
    transform: rotate(0deg) scale(0.8);
  }

  to {
    transform: rotate(1080deg) scale(1);
  }
}

.reward-small-title {
  margin: 6px 0 0;
  color: #fde68a;
  font-size: 18px;
  font-weight: bold;
  letter-spacing: 1px;
}

.reward-main-title {
  margin: 10px 0 20px;
  font-size: 46px;
  color: white;
  text-shadow: 0 0 18px rgba(255, 255, 255, 0.45);
}

.reward-detail-box {
  margin-top: 16px;
  padding: 18px;
  border-radius: 18px;
  text-align: left;
  background: rgba(15, 23, 42, 0.52);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.reward-detail-box h3 {
  margin: 0 0 8px;
  color: #bfdbfe;
  font-size: 20px;
}

.reward-detail-box p {
  margin: 0;
  color: #f8fafc;
  font-size: 17px;
  line-height: 1.6;
}

.use-box {
  background: rgba(22, 101, 52, 0.42);
  border-color: rgba(134, 239, 172, 0.35);
}

.reward-btn {
  width: 100%;
  margin-top: 24px;
  font-size: 20px;
  padding: 16px;
}

.game-over-card {
  width: 520px;
  padding: 38px;
  background:
    radial-gradient(circle at top, rgba(248, 113, 113, 0.6), transparent 35%),
    linear-gradient(135deg, #450a0a, #7f1d1d, #312e81);
}

.game-over-icon {
  width: 115px;
  height: 115px;
  display: grid;
  place-items: center;
  margin: 0 auto 12px;
  border-radius: 50%;
  background: rgba(15, 23, 42, 0.75);
  font-size: 54px;
  box-shadow: 0 0 35px rgba(248, 113, 113, 0.65);
  animation: gameOverPop 0.45s ease-out;
}

.game-over-small {
  margin: 0;
  color: #fecaca;
  font-weight: bold;
  letter-spacing: 2px;
}

.game-over-card h2 {
  margin: 8px 0;
  color: white;
  font-size: 50px;
  text-shadow: 0 0 22px rgba(255, 255, 255, 0.35);
}

.game-over-subtitle {
  margin: 0;
  color: #fee2e2;
  font-size: 16px;
}

.game-over-score-box {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 14px;
  margin-top: 24px;
}

.game-over-score-box div {
  padding: 16px;
  border-radius: 18px;
  background: rgba(15, 23, 42, 0.55);
  border: 1px solid rgba(255, 255, 255, 0.18);
}

.game-over-score-box span {
  display: block;
  color: #fecaca;
  font-size: 14px;
}

.game-over-score-box strong {
  display: block;
  margin-top: 6px;
  color: #ffffff;
  font-size: 42px;
}

.game-over-buttons {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 14px;
  margin-top: 24px;
}

.game-over-btn {
  font-size: 18px;
  padding: 15px;
}

@keyframes gameOverPop {
  from {
    transform: scale(0.4);
    opacity: 0;
  }

  to {
    transform: scale(1);
    opacity: 1;
  }
}

@media (max-width: 1150px) {
  .page {
    overflow: auto;
  }

  .game-shell {
    width: 100%;
  }

  .header {
    flex-direction: column;
    align-items: flex-start;
  }

  .top-boxes {
    width: 100%;
    flex-wrap: wrap;
  }

  .select-box,
  .status-box {
    flex: 1;
    width: auto;
    min-width: 140px;
  }

  .mode-showcase {
    grid-template-columns: 1fr;
  }

  .game-layout {
    flex-direction: column;
  }

  .board {
    width: 100%;
    aspect-ratio: 1 / 1;
    height: auto;
  }

  .panel {
    width: 100%;
  }

  .board-title {
    flex-wrap: wrap;
  }

  .bottom-dashboard {
    grid-template-columns: 1fr;
  }

  .big-reward-card,
  .game-over-card,
  .avatar-modal {
    width: 92%;
  }

  .avatar-modal-body {
    grid-template-columns: 1fr;
  }
}
</style>