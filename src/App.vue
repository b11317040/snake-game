<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type Position = { x: number; y: number }
type GameStatus = 'ready' | 'playing' | 'paused' | 'over' | 'reward'
type Language =
  | 'zhTw'
  | 'en'
  | 'ja'
  | 'ko'
  | 'es'
  | 'fr'
  | 'de'
  | 'pt'
  | 'ru'
  | 'ar'
  | 'hi'
  | 'id'
  | 'vi'
  | 'th'

type RewardKey =
  | 'life1'
  | 'life2'
  | 'shorten'
  | 'removeObstacle'
  | 'clearBomb'
  | 'slow'
  | 'shield'
  | 'hideObstacle'

const boardSize = 20
const baseSpeed = 140
const skillDurationMs = 10000
const respawnProtectionMs = 900

const snake = ref<Position[]>([{ x: 10, y: 10 }])
const food = ref<Position>({ x: 5, y: 5 })
const obstacles = ref<Position[]>([])
const bombs = ref<Position[]>([])

const direction = ref<Position>({ x: 1, y: 0 })
const nextDirection = ref<Position>({ x: 1, y: 0 })

const score = ref(0)
const bestScore = ref(0)
const lives = ref(1)
const level = ref(1)
const speed = ref(baseSpeed)
const status = ref<GameStatus>('ready')
const language = ref<Language>('zhTw')

const lastDifficultyScore = ref(0)
const lastRewardScore = ref(0)

const currentSkillKey = ref<RewardKey | null>(null)
const rewardKey = ref<RewardKey | null>(null)

const shieldUntil = ref(0)
const slowUntil = ref(0)
const hideObstacleUntil = ref(0)
const invincibleUntil = ref(0)
const now = ref(Date.now())

let timer: number | undefined
let clockTimer: number | undefined
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

const text = {
  zhTw: {
    project: 'Vue 專案',
    title: '進階彩色貪食蛇',
    description: '吃食物、躲炸彈、避開障礙，挑戰更高分數。',
    language: '切換語言',
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
    legend: '圖示說明',
    noSkill: '無',
    allSkills: '可能出現的技能',
    start: '開始',
    pause: '暫停',
    resume: '繼續',
    restart: '重新開始',
    exit: '退出',
    readyMessage: '按下開始按鈕即可開始遊戲。',
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
    ruleFood: '吃到粉紅色食物 = +1 分。',
    ruleHazard: '撞牆、撞自己、撞紫色障礙或藍色炸彈會扣 1 顆生命，蛇身長度不會減少。',
    ruleLevel: '每 5 分進到下一關，速度會變快，障礙與炸彈也會增加。',
    ruleReward: '每 10 分會觸發技能輪盤，技能會自動生效。',
    activeEffects: '技能倒數',
    noActiveEffect: '目前沒有技能效果',
    legendFood: '食物',
    legendObstacle: '障礙',
    legendBomb: '炸彈',
    secondPerStep: '秒 / 格',
    second: '秒',
  },
  en: {
    project: 'Vue Project',
    title: 'Advanced Color Snake',
    description: 'Eat food, dodge bombs, avoid obstacles, and chase a higher score.',
    language: 'Language',
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
    speed: 'Snake Speed',
    skill: 'Current Skill',
    nextReward: 'Next Reward',
    obstacles: 'Obstacles',
    bombs: 'Bombs',
    legend: 'Legend',
    noSkill: 'None',
    allSkills: 'Possible Skills',
    start: 'Start',
    pause: 'Pause',
    resume: 'Resume',
    restart: 'Restart',
    exit: 'Exit',
    readyMessage: 'Press Start to begin the game.',
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
    ruleFood: 'Eat pink food = +1 point.',
    ruleHazard: 'Wall, body, purple obstacle, or blue bomb costs 1 life. Snake length does not shrink.',
    ruleLevel: 'Every 5 points takes you to the next level. Speed, obstacles, and bombs increase.',
    ruleReward: 'Every 10 points triggers the reward wheel. Skills activate automatically.',
    activeEffects: 'Effect Timer',
    noActiveEffect: 'No active effect',
    legendFood: 'Food',
    legendObstacle: 'Obstacle',
    legendBomb: 'Bomb',
    secondPerStep: 's / step',
    second: 's',
  },
  ja: {
    project: 'Vue プロジェクト',
    title: '上級カラースネーク',
    description: '食べ物を食べ、爆弾と障害物を避けて高得点を狙おう。',
    language: '言語',
    status: '状態',
    ready: '準備中',
    playing: 'プレイ中',
    paused: '一時停止',
    over: 'ゲーム終了',
    reward: '報酬ルーレット',
    boardTitle: '20 × 20 ゲームボード',
    score: 'スコア',
    finalScore: '今回のスコア',
    bestScore: 'ベスト',
    lives: 'ライフ',
    level: 'レベル',
    speed: '速度',
    skill: '現在のスキル',
    nextReward: '次の報酬',
    obstacles: '障害物',
    bombs: '爆弾',
    legend: '凡例',
    noSkill: 'なし',
    allSkills: '出現するスキル',
    start: 'スタート',
    pause: '一時停止',
    resume: '再開',
    restart: 'リスタート',
    exit: '終了',
    readyMessage: 'スタートを押して開始してください。',
    pausedMessage: '一時停止中です。再開またはスペースキーを押してください。',
    overMessage: 'ゲーム終了！もう一度挑戦できます。',
    gameOverTitle: 'ゲーム終了',
    gameOverSubtitle: 'もう一度挑戦して最高得点を更新しよう！',
    rewardTitle: '報酬ルーレット',
    rewardButton: '受け取って続ける',
    skillEffect: 'スキル効果',
    howToUse: '使い方',
    controls: '操作方法',
    space: 'スペース：一時停止 / 再開',
    rules: 'ルール',
    ruleFood: 'ピンクの食べ物 = +1点。',
    ruleHazard: '壁・体・紫の障害物・青い爆弾に当たるとライフが1減ります。ヘビの長さは減りません。',
    ruleLevel: '5点ごとに次のレベルへ進み、速度・障害物・爆弾が増えます。',
    ruleReward: '10点ごとに報酬ルーレットが発動し、スキルは自動で有効になります。',
    activeEffects: '効果時間',
    noActiveEffect: '有効な効果なし',
    legendFood: '食べ物',
    legendObstacle: '障害物',
    legendBomb: '爆弾',
    secondPerStep: '秒 / マス',
    second: '秒',
  },
  ko: {
    project: 'Vue 프로젝트',
    title: '고급 컬러 스네이크',
    description: '음식을 먹고 폭탄과 장애물을 피해 높은 점수에 도전하세요.',
    language: '언어',
    status: '상태',
    ready: '준비',
    playing: '게임 중',
    paused: '일시 정지',
    over: '게임 오버',
    reward: '보상 룰렛',
    boardTitle: '20 × 20 게임 보드',
    score: '점수',
    finalScore: '최종 점수',
    bestScore: '최고 점수',
    lives: '생명',
    level: '레벨',
    speed: '속도',
    skill: '현재 스킬',
    nextReward: '다음 보상',
    obstacles: '장애물',
    bombs: '폭탄',
    legend: '범례',
    noSkill: '없음',
    allSkills: '가능한 스킬',
    start: '시작',
    pause: '일시 정지',
    resume: '계속',
    restart: '다시 시작',
    exit: '나가기',
    readyMessage: '시작 버튼을 눌러 게임을 시작하세요.',
    pausedMessage: '게임이 일시 정지되었습니다. 계속 또는 Space를 누르세요.',
    overMessage: '게임 오버! 다시 도전할 수 있습니다.',
    gameOverTitle: '게임 오버',
    gameOverSubtitle: '다시 도전해서 최고 점수를 갱신하세요!',
    rewardTitle: '보상 룰렛',
    rewardButton: '받고 계속하기',
    skillEffect: '스킬 효과',
    howToUse: '사용 방법',
    controls: '조작 방법',
    space: 'Space: 일시 정지 / 계속',
    rules: '규칙',
    ruleFood: '분홍색 음식 = +1점.',
    ruleHazard: '벽, 몸, 보라색 장애물, 파란 폭탄에 닿으면 생명 1개 감소. 뱀 길이는 줄어들지 않습니다.',
    ruleLevel: '5점마다 다음 레벨로 이동하며 속도, 장애물, 폭탄이 증가합니다.',
    ruleReward: '10점마다 보상 룰렛이 나오며 스킬은 자동 적용됩니다.',
    activeEffects: '효과 시간',
    noActiveEffect: '활성 효과 없음',
    legendFood: '음식',
    legendObstacle: '장애물',
    legendBomb: '폭탄',
    secondPerStep: '초 / 칸',
    second: '초',
  },
  es: {
    project: 'Proyecto Vue',
    title: 'Snake Colorido Avanzado',
    description: 'Come comida, esquiva bombas y evita obstáculos.',
    language: 'Idioma',
    status: 'Estado',
    ready: 'Listo',
    playing: 'Jugando',
    paused: 'Pausado',
    over: 'Fin',
    reward: 'Ruleta',
    boardTitle: 'Tablero 20 × 20',
    score: 'Puntos',
    finalScore: 'Puntos finales',
    bestScore: 'Mejor',
    lives: 'Vidas',
    level: 'Nivel',
    speed: 'Velocidad',
    skill: 'Habilidad',
    nextReward: 'Próxima recompensa',
    obstacles: 'Obstáculos',
    bombs: 'Bombas',
    legend: 'Leyenda',
    noSkill: 'Ninguna',
    allSkills: 'Habilidades posibles',
    start: 'Iniciar',
    pause: 'Pausar',
    resume: 'Continuar',
    restart: 'Reiniciar',
    exit: 'Salir',
    readyMessage: 'Presiona Iniciar para comenzar.',
    pausedMessage: 'Juego pausado. Presiona Continuar o Espacio.',
    overMessage: 'Fin del juego. Puedes reiniciar.',
    gameOverTitle: 'Fin del juego',
    gameOverSubtitle: 'Intenta otra vez y supera tu mejor puntuación.',
    rewardTitle: 'Ruleta de Recompensa',
    rewardButton: 'Tomar y continuar',
    skillEffect: 'Efecto',
    howToUse: 'Uso',
    controls: 'Controles',
    space: 'Espacio: Pausar / Continuar',
    rules: 'Reglas',
    ruleFood: 'Comida rosa = +1 punto.',
    ruleHazard: 'Pared, cuerpo, obstáculo morado o bomba azul cuesta 1 vida. La serpiente no se acorta.',
    ruleLevel: 'Cada 5 puntos pasas al siguiente nivel. Aumentan velocidad, obstáculos y bombas.',
    ruleReward: 'Cada 10 puntos aparece la ruleta. Las habilidades se activan solas.',
    activeEffects: 'Tiempo de efecto',
    noActiveEffect: 'Sin efecto activo',
    legendFood: 'Comida',
    legendObstacle: 'Obstáculo',
    legendBomb: 'Bomba',
    secondPerStep: 's / paso',
    second: 's',
  },
  fr: {
    project: 'Projet Vue',
    title: 'Snake Coloré Avancé',
    description: 'Mange, évite les bombes et les obstacles.',
    language: 'Langue',
    status: 'État',
    ready: 'Prêt',
    playing: 'En jeu',
    paused: 'Pause',
    over: 'Terminé',
    reward: 'Roue',
    boardTitle: 'Plateau 20 × 20',
    score: 'Score',
    finalScore: 'Score final',
    bestScore: 'Meilleur',
    lives: 'Vies',
    level: 'Niveau',
    speed: 'Vitesse',
    skill: 'Compétence',
    nextReward: 'Prochaine récompense',
    obstacles: 'Obstacles',
    bombs: 'Bombes',
    legend: 'Légende',
    noSkill: 'Aucune',
    allSkills: 'Compétences possibles',
    start: 'Démarrer',
    pause: 'Pause',
    resume: 'Reprendre',
    restart: 'Recommencer',
    exit: 'Quitter',
    readyMessage: 'Appuie sur Démarrer.',
    pausedMessage: 'Jeu en pause. Appuie sur Reprendre ou Espace.',
    overMessage: 'Partie terminée. Tu peux recommencer.',
    gameOverTitle: 'Partie terminée',
    gameOverSubtitle: 'Rejoue pour battre ton meilleur score.',
    rewardTitle: 'Roue de Récompense',
    rewardButton: 'Prendre et continuer',
    skillEffect: 'Effet',
    howToUse: 'Utilisation',
    controls: 'Contrôles',
    space: 'Espace : Pause / Reprendre',
    rules: 'Règles',
    ruleFood: 'Nourriture rose = +1 point.',
    ruleHazard: 'Mur, corps, obstacle violet ou bombe bleue coûte 1 vie. Le serpent ne raccourcit pas.',
    ruleLevel: 'Tous les 5 points, tu passes au niveau suivant. Vitesse, obstacles et bombes augmentent.',
    ruleReward: 'Tous les 10 points, la roue apparaît. Les compétences sont automatiques.',
    activeEffects: 'Temps effet',
    noActiveEffect: 'Aucun effet actif',
    legendFood: 'Nourriture',
    legendObstacle: 'Obstacle',
    legendBomb: 'Bombe',
    secondPerStep: 's / case',
    second: 's',
  },
  de: {
    project: 'Vue Projekt',
    title: 'Fortgeschrittenes Snake',
    description: 'Iss Futter, meide Bomben und Hindernisse.',
    language: 'Sprache',
    status: 'Status',
    ready: 'Bereit',
    playing: 'Spiel',
    paused: 'Pause',
    over: 'Vorbei',
    reward: 'Rad',
    boardTitle: '20 × 20 Spielfeld',
    score: 'Punkte',
    finalScore: 'Endpunkte',
    bestScore: 'Bestwert',
    lives: 'Leben',
    level: 'Level',
    speed: 'Tempo',
    skill: 'Skill',
    nextReward: 'Nächste Belohnung',
    obstacles: 'Hindernisse',
    bombs: 'Bomben',
    legend: 'Legende',
    noSkill: 'Kein',
    allSkills: 'Mögliche Skills',
    start: 'Start',
    pause: 'Pause',
    resume: 'Fortsetzen',
    restart: 'Neustart',
    exit: 'Beenden',
    readyMessage: 'Drücke Start.',
    pausedMessage: 'Spiel pausiert. Drücke Fortsetzen oder Leertaste.',
    overMessage: 'Spiel vorbei. Starte neu.',
    gameOverTitle: 'Spiel vorbei',
    gameOverSubtitle: 'Versuche es erneut und schlage deinen Bestwert.',
    rewardTitle: 'Belohnungsrad',
    rewardButton: 'Nehmen und weiter',
    skillEffect: 'Effekt',
    howToUse: 'Benutzung',
    controls: 'Steuerung',
    space: 'Leertaste: Pause / Fortsetzen',
    rules: 'Regeln',
    ruleFood: 'Pinkes Futter = +1 Punkt.',
    ruleHazard: 'Wand, Körper, lila Hindernis oder blaue Bombe kostet 1 Leben. Die Schlange wird nicht kürzer.',
    ruleLevel: 'Alle 5 Punkte kommst du ins nächste Level. Tempo, Hindernisse und Bomben steigen.',
    ruleReward: 'Alle 10 Punkte erscheint das Rad. Skills wirken automatisch.',
    activeEffects: 'Effektzeit',
    noActiveEffect: 'Kein aktiver Effekt',
    legendFood: 'Futter',
    legendObstacle: 'Hindernis',
    legendBomb: 'Bombe',
    secondPerStep: 's / Feld',
    second: 's',
  },
  pt: {
    project: 'Projeto Vue',
    title: 'Snake Colorido Avançado',
    description: 'Coma comida, desvie de bombas e obstáculos.',
    language: 'Idioma',
    status: 'Estado',
    ready: 'Pronto',
    playing: 'Jogando',
    paused: 'Pausado',
    over: 'Fim',
    reward: 'Roleta',
    boardTitle: 'Tabuleiro 20 × 20',
    score: 'Pontos',
    finalScore: 'Pontuação final',
    bestScore: 'Melhor',
    lives: 'Vidas',
    level: 'Nível',
    speed: 'Velocidade',
    skill: 'Habilidade',
    nextReward: 'Próxima recompensa',
    obstacles: 'Obstáculos',
    bombs: 'Bombas',
    legend: 'Legenda',
    noSkill: 'Nenhuma',
    allSkills: 'Habilidades possíveis',
    start: 'Iniciar',
    pause: 'Pausar',
    resume: 'Continuar',
    restart: 'Reiniciar',
    exit: 'Sair',
    readyMessage: 'Pressione Iniciar.',
    pausedMessage: 'Jogo pausado. Pressione Continuar ou Espaço.',
    overMessage: 'Fim de jogo. Tente novamente.',
    gameOverTitle: 'Fim de jogo',
    gameOverSubtitle: 'Tente novamente e supere sua melhor pontuação.',
    rewardTitle: 'Roleta de Recompensa',
    rewardButton: 'Receber e continuar',
    skillEffect: 'Efeito',
    howToUse: 'Como usar',
    controls: 'Controles',
    space: 'Espaço: Pausar / Continuar',
    rules: 'Regras',
    ruleFood: 'Comida rosa = +1 ponto.',
    ruleHazard: 'Parede, corpo, obstáculo roxo ou bomba azul custa 1 vida. A cobra não fica menor.',
    ruleLevel: 'A cada 5 pontos você vai para a próxima fase. Velocidade, obstáculos e bombas aumentam.',
    ruleReward: 'A cada 10 pontos aparece a roleta. Habilidades são automáticas.',
    activeEffects: 'Tempo do efeito',
    noActiveEffect: 'Nenhum efeito ativo',
    legendFood: 'Comida',
    legendObstacle: 'Obstáculo',
    legendBomb: 'Bomba',
    secondPerStep: 's / passo',
    second: 's',
  },
  ru: {
    project: 'Проект Vue',
    title: 'Продвинутая Змейка',
    description: 'Ешь еду, избегай бомб и препятствий.',
    language: 'Язык',
    status: 'Статус',
    ready: 'Готово',
    playing: 'Игра',
    paused: 'Пауза',
    over: 'Конец',
    reward: 'Рулетка',
    boardTitle: 'Поле 20 × 20',
    score: 'Счёт',
    finalScore: 'Итоговый счёт',
    bestScore: 'Лучший',
    lives: 'Жизни',
    level: 'Уровень',
    speed: 'Скорость',
    skill: 'Навык',
    nextReward: 'Следующая награда',
    obstacles: 'Препятствия',
    bombs: 'Бомбы',
    legend: 'Легенда',
    noSkill: 'Нет',
    allSkills: 'Возможные навыки',
    start: 'Старт',
    pause: 'Пауза',
    resume: 'Продолжить',
    restart: 'Заново',
    exit: 'Выход',
    readyMessage: 'Нажмите Старт.',
    pausedMessage: 'Пауза. Нажмите Продолжить или Пробел.',
    overMessage: 'Игра окончена. Начните снова.',
    gameOverTitle: 'Игра окончена',
    gameOverSubtitle: 'Попробуй снова и побей лучший счёт.',
    rewardTitle: 'Рулетка наград',
    rewardButton: 'Получить и продолжить',
    skillEffect: 'Эффект',
    howToUse: 'Как использовать',
    controls: 'Управление',
    space: 'Пробел: Пауза / Продолжить',
    rules: 'Правила',
    ruleFood: 'Розовая еда = +1 очко.',
    ruleHazard: 'Стена, тело, фиолетовое препятствие или синяя бомба отнимает 1 жизнь. Змейка не становится короче.',
    ruleLevel: 'Каждые 5 очков ты переходишь на следующий уровень. Скорость, препятствия и бомбы растут.',
    ruleReward: 'Каждые 10 очков появляется рулетка. Навыки активируются автоматически.',
    activeEffects: 'Таймер эффекта',
    noActiveEffect: 'Нет активного эффекта',
    legendFood: 'Еда',
    legendObstacle: 'Препятствие',
    legendBomb: 'Бомба',
    secondPerStep: 'с / шаг',
    second: 'с',
  },
  ar: {
    project: 'مشروع Vue',
    title: 'لعبة الثعبان المتقدمة',
    description: 'كل الطعام وتجنب القنابل والعوائق.',
    language: 'اللغة',
    status: 'الحالة',
    ready: 'جاهز',
    playing: 'قيد اللعب',
    paused: 'إيقاف',
    over: 'انتهت',
    reward: 'العجلة',
    boardTitle: 'لوحة 20 × 20',
    score: 'النقاط',
    finalScore: 'النتيجة النهائية',
    bestScore: 'الأفضل',
    lives: 'الحياة',
    level: 'المستوى',
    speed: 'السرعة',
    skill: 'المهارة',
    nextReward: 'المكافأة التالية',
    obstacles: 'العوائق',
    bombs: 'القنابل',
    legend: 'الرموز',
    noSkill: 'لا يوجد',
    allSkills: 'المهارات الممكنة',
    start: 'ابدأ',
    pause: 'إيقاف',
    resume: 'متابعة',
    restart: 'إعادة',
    exit: 'خروج',
    readyMessage: 'اضغط ابدأ.',
    pausedMessage: 'اللعبة متوقفة. اضغط متابعة أو المسافة.',
    overMessage: 'انتهت اللعبة. حاول مرة أخرى.',
    gameOverTitle: 'انتهت اللعبة',
    gameOverSubtitle: 'حاول مرة أخرى وحقق نتيجة أفضل.',
    rewardTitle: 'عجلة المكافآت',
    rewardButton: 'استلام ومتابعة',
    skillEffect: 'التأثير',
    howToUse: 'طريقة الاستخدام',
    controls: 'التحكم',
    space: 'المسافة: إيقاف / متابعة',
    rules: 'القواعد',
    ruleFood: 'الطعام الوردي = +1 نقطة.',
    ruleHazard: 'الجدار أو الجسم أو العائق البنفسجي أو القنبلة الزرقاء تنقص حياة واحدة ولا يقل طول الثعبان.',
    ruleLevel: 'كل 5 نقاط تنتقل إلى المرحلة التالية وتزيد السرعة والعوائق والقنابل.',
    ruleReward: 'كل 10 نقاط تظهر العجلة. المهارات تعمل تلقائياً.',
    activeEffects: 'وقت التأثير',
    noActiveEffect: 'لا يوجد تأثير فعال',
    legendFood: 'طعام',
    legendObstacle: 'عائق',
    legendBomb: 'قنبلة',
    secondPerStep: 'ث / خطوة',
    second: 'ث',
  },
  hi: {
    project: 'Vue प्रोजेक्ट',
    title: 'एडवांस स्नेक गेम',
    description: 'खाना खाएँ, बम और बाधाओं से बचें।',
    language: 'भाषा',
    status: 'स्थिति',
    ready: 'तैयार',
    playing: 'खेल',
    paused: 'रुका',
    over: 'समाप्त',
    reward: 'इनाम',
    boardTitle: '20 × 20 बोर्ड',
    score: 'स्कोर',
    finalScore: 'अंतिम स्कोर',
    bestScore: 'सर्वश्रेष्ठ',
    lives: 'जीवन',
    level: 'लेवल',
    speed: 'गति',
    skill: 'कौशल',
    nextReward: 'अगला इनाम',
    obstacles: 'बाधाएँ',
    bombs: 'बम',
    legend: 'चिह्न',
    noSkill: 'कोई नहीं',
    allSkills: 'संभावित कौशल',
    start: 'शुरू',
    pause: 'रोकें',
    resume: 'जारी',
    restart: 'फिर शुरू',
    exit: 'बाहर',
    readyMessage: 'Start दबाएँ।',
    pausedMessage: 'खेल रुका है। Resume या Space दबाएँ।',
    overMessage: 'गेम समाप्त। फिर से कोशिश करें।',
    gameOverTitle: 'गेम समाप्त',
    gameOverSubtitle: 'फिर खेलें और अपना सर्वश्रेष्ठ स्कोर तोड़ें।',
    rewardTitle: 'इनाम चक्र',
    rewardButton: 'लेकर जारी रखें',
    skillEffect: 'कौशल प्रभाव',
    howToUse: 'कैसे उपयोग करें',
    controls: 'नियंत्रण',
    space: 'Space: रोकें / जारी',
    rules: 'नियम',
    ruleFood: 'गुलाबी खाना = +1 अंक।',
    ruleHazard: 'दीवार, शरीर, बैंगनी बाधा या नीला बम 1 जीवन घटाता है। साँप की लंबाई कम नहीं होती।',
    ruleLevel: 'हर 5 अंक पर आप अगले लेवल में जाते हैं। गति, बाधाएँ और बम बढ़ते हैं।',
    ruleReward: 'हर 10 अंक पर इनाम चक्र आता है। कौशल अपने आप सक्रिय होते हैं।',
    activeEffects: 'प्रभाव समय',
    noActiveEffect: 'कोई सक्रिय प्रभाव नहीं',
    legendFood: 'खाना',
    legendObstacle: 'बाधा',
    legendBomb: 'बम',
    secondPerStep: 'से / कदम',
    second: 'से',
  },
  id: {
    project: 'Proyek Vue',
    title: 'Snake Warna Lanjutan',
    description: 'Makan makanan, hindari bom dan rintangan.',
    language: 'Bahasa',
    status: 'Status',
    ready: 'Siap',
    playing: 'Bermain',
    paused: 'Jeda',
    over: 'Selesai',
    reward: 'Roda Hadiah',
    boardTitle: 'Papan 20 × 20',
    score: 'Skor',
    finalScore: 'Skor akhir',
    bestScore: 'Terbaik',
    lives: 'Nyawa',
    level: 'Level',
    speed: 'Kecepatan',
    skill: 'Skill',
    nextReward: 'Hadiah berikutnya',
    obstacles: 'Rintangan',
    bombs: 'Bom',
    legend: 'Legenda',
    noSkill: 'Tidak ada',
    allSkills: 'Skill yang mungkin',
    start: 'Mulai',
    pause: 'Jeda',
    resume: 'Lanjut',
    restart: 'Ulangi',
    exit: 'Keluar',
    readyMessage: 'Tekan Mulai.',
    pausedMessage: 'Game dijeda. Tekan Lanjut atau Spasi.',
    overMessage: 'Game selesai. Coba lagi.',
    gameOverTitle: 'Game selesai',
    gameOverSubtitle: 'Coba lagi dan kalahkan skor terbaikmu.',
    rewardTitle: 'Roda Hadiah',
    rewardButton: 'Ambil dan lanjut',
    skillEffect: 'Efek Skill',
    howToUse: 'Cara Pakai',
    controls: 'Kontrol',
    space: 'Spasi: Jeda / Lanjut',
    rules: 'Aturan',
    ruleFood: 'Makanan merah muda = +1 poin.',
    ruleHazard: 'Dinding, tubuh, rintangan ungu, atau bom biru mengurangi 1 nyawa. Panjang ular tidak berkurang.',
    ruleLevel: 'Setiap 5 poin masuk ke level berikutnya. Kecepatan, rintangan, dan bom bertambah.',
    ruleReward: 'Setiap 10 poin roda hadiah muncul. Skill aktif otomatis.',
    activeEffects: 'Waktu Efek',
    noActiveEffect: 'Tidak ada efek aktif',
    legendFood: 'Makanan',
    legendObstacle: 'Rintangan',
    legendBomb: 'Bom',
    secondPerStep: 'd / langkah',
    second: 'd',
  },
  vi: {
    project: 'Dự án Vue',
    title: 'Snake Nâng Cao',
    description: 'Ăn thức ăn, né bom và vật cản.',
    language: 'Ngôn ngữ',
    status: 'Trạng thái',
    ready: 'Sẵn sàng',
    playing: 'Đang chơi',
    paused: 'Tạm dừng',
    over: 'Kết thúc',
    reward: 'Vòng quay',
    boardTitle: 'Bảng 20 × 20',
    score: 'Điểm',
    finalScore: 'Điểm cuối',
    bestScore: 'Cao nhất',
    lives: 'Mạng',
    level: 'Cấp',
    speed: 'Tốc độ',
    skill: 'Kỹ năng',
    nextReward: 'Thưởng tiếp',
    obstacles: 'Vật cản',
    bombs: 'Bom',
    legend: 'Chú thích',
    noSkill: 'Không có',
    allSkills: 'Kỹ năng có thể xuất hiện',
    start: 'Bắt đầu',
    pause: 'Tạm dừng',
    resume: 'Tiếp tục',
    restart: 'Chơi lại',
    exit: 'Thoát',
    readyMessage: 'Nhấn Bắt đầu.',
    pausedMessage: 'Đã tạm dừng. Nhấn Tiếp tục hoặc Space.',
    overMessage: 'Kết thúc. Hãy thử lại.',
    gameOverTitle: 'Kết thúc',
    gameOverSubtitle: 'Chơi lại để vượt điểm cao nhất.',
    rewardTitle: 'Vòng quay thưởng',
    rewardButton: 'Nhận và tiếp tục',
    skillEffect: 'Hiệu ứng',
    howToUse: 'Cách dùng',
    controls: 'Điều khiển',
    space: 'Space: Tạm dừng / Tiếp tục',
    rules: 'Luật',
    ruleFood: 'Thức ăn màu hồng = +1 điểm.',
    ruleHazard: 'Tường, thân, vật cản tím hoặc bom xanh làm mất 1 mạng. Chiều dài rắn không giảm.',
    ruleLevel: 'Mỗi 5 điểm sang cấp tiếp theo. Tốc độ, vật cản và bom tăng lên.',
    ruleReward: 'Mỗi 10 điểm có vòng quay. Kỹ năng tự động kích hoạt.',
    activeEffects: 'Thời gian hiệu ứng',
    noActiveEffect: 'Không có hiệu ứng',
    legendFood: 'Thức ăn',
    legendObstacle: 'Vật cản',
    legendBomb: 'Bom',
    secondPerStep: 'giây / ô',
    second: 'giây',
  },
  th: {
    project: 'โปรเจกต์ Vue',
    title: 'เกมงูขั้นสูง',
    description: 'กินอาหาร หลบระเบิดและสิ่งกีดขวาง',
    language: 'ภาษา',
    status: 'สถานะ',
    ready: 'พร้อม',
    playing: 'เล่น',
    paused: 'หยุด',
    over: 'จบ',
    reward: 'วงล้อ',
    boardTitle: 'กระดาน 20 × 20',
    score: 'คะแนน',
    finalScore: 'คะแนนสุดท้าย',
    bestScore: 'สูงสุด',
    lives: 'ชีวิต',
    level: 'ระดับ',
    speed: 'ความเร็ว',
    skill: 'สกิล',
    nextReward: 'รางวัลถัดไป',
    obstacles: 'สิ่งกีดขวาง',
    bombs: 'ระเบิด',
    legend: 'สัญลักษณ์',
    noSkill: 'ไม่มี',
    allSkills: 'สกิลที่อาจได้',
    start: 'เริ่ม',
    pause: 'หยุด',
    resume: 'เล่นต่อ',
    restart: 'เริ่มใหม่',
    exit: 'ออก',
    readyMessage: 'กดเริ่ม',
    pausedMessage: 'หยุดชั่วคราว กดเล่นต่อหรือ Space',
    overMessage: 'จบเกม ลองใหม่ได้',
    gameOverTitle: 'จบเกม',
    gameOverSubtitle: 'ลองใหม่เพื่อทำคะแนนสูงสุด',
    rewardTitle: 'วงล้อรางวัล',
    rewardButton: 'รับและเล่นต่อ',
    skillEffect: 'ผลของสกิล',
    howToUse: 'วิธีใช้',
    controls: 'ควบคุม',
    space: 'Space: หยุด / เล่นต่อ',
    rules: 'กติกา',
    ruleFood: 'อาหารสีชมพู = +1 คะแนน',
    ruleHazard: 'ชนกำแพง ตัวเอง สิ่งกีดขวางสีม่วง หรือระเบิดสีน้ำเงินเสีย 1 ชีวิต ความยาวงูไม่ลดลง',
    ruleLevel: 'ทุก 5 คะแนนจะเข้าสู่ด่านถัดไป ความเร็ว สิ่งกีดขวาง และระเบิดจะเพิ่มขึ้น',
    ruleReward: 'ทุก 10 คะแนนจะมีวงล้อรางวัล สกิลทำงานอัตโนมัติ',
    activeEffects: 'เวลาสกิล',
    noActiveEffect: 'ไม่มีสกิลทำงาน',
    legendFood: 'อาหาร',
    legendObstacle: 'สิ่งกีดขวาง',
    legendBomb: 'ระเบิด',
    secondPerStep: 'วิ / ช่อง',
    second: 'วิ',
  },
}

const rewardTexts = {
  zhTw: {
    life1: ['生命 +1', '生命 +1，之後撞到危險物時可以多撐一次。', '自動生效，地圖上方會增加一顆紅色愛心。'],
    life2: ['生命 +2', '生命 +2，後面關卡容錯率更高。', '自動生效，地圖上方會增加兩顆紅色愛心。'],
    shorten: ['蛇身 -3', '蛇的身體縮短 3 格，轉彎和閃避更容易。', '自動生效，身體會立刻變短。'],
    removeObstacle: ['移除 3 個障礙', '移除場上 3 個紫色障礙物。', '自動生效，部分障礙會立刻消失。'],
    clearBomb: ['清除所有炸彈', '清除場上所有藍色炸彈。', '自動生效，所有炸彈會消失。'],
    slow: ['緩速 10 秒', '接下來 10 秒蛇的速度變慢。', '自動生效，趁這段時間吃食物或找安全路線。'],
    shield: ['護盾 10 秒', '10 秒內碰到障礙或炸彈不會死亡，會把它清掉。', '自動生效，蛇頭變成金色時可以撞掉危險物。'],
    hideObstacle: ['障礙失效 10 秒', '10 秒內紫色障礙暫時失效，可以安全通過。', '自動生效，趁障礙失效時快速移動。'],
  },
  en: {
    life1: ['Life +1', 'Gain 1 extra life for future mistakes.', 'Works automatically. One red heart is added.'],
    life2: ['Life +2', 'Gain 2 extra lives for harder levels.', 'Works automatically. Two red hearts are added.'],
    shorten: ['Snake -3', 'Shortens the snake by 3 blocks.', 'Works automatically and makes turning easier.'],
    removeObstacle: ['Remove 3 Obstacles', 'Removes 3 purple obstacles.', 'Works automatically. Some obstacles disappear.'],
    clearBomb: ['Clear All Bombs', 'Removes all blue bombs.', 'Works automatically. All bombs disappear.'],
    slow: ['Slow 10s', 'The snake slows down for 10 seconds.', 'Use this time to eat food or find a safe route.'],
    shield: ['Shield 10s', 'For 10 seconds, obstacles and bombs are destroyed instead of hurting you.', 'Works automatically. Golden head means shield is active.'],
    hideObstacle: ['Disable Obstacles 10s', 'Purple obstacles are disabled for 10 seconds.', 'Move through obstacles safely while active.'],
  },
  ja: {
    life1: ['ライフ +1', 'ミスしても1回多く耐えられます。', '自動で発動し、赤いハートが1つ増えます。'],
    life2: ['ライフ +2', '難しいレベルに備えてライフが2つ増えます。', '自動で発動し、赤いハートが2つ増えます。'],
    shorten: ['体 -3', 'ヘビの体が3マス短くなります。', '自動で発動し、曲がりやすくなります。'],
    removeObstacle: ['障害物を3つ除去', '紫の障害物を3つ消します。', '自動で発動します。'],
    clearBomb: ['爆弾を全消去', '青い爆弾をすべて消します。', '自動で発動します。'],
    slow: ['低速 10秒', '10秒間ヘビが遅くなります。', '安全ルートを探す時間に使えます。'],
    shield: ['シールド 10秒', '10秒間、障害物や爆弾を壊せます。', '金色の頭の間は有効です。'],
    hideObstacle: ['障害物無効 10秒', '10秒間、紫の障害物が無効になります。', 'その間に安全に通過できます。'],
  },
  ko: {
    life1: ['생명 +1', '실수해도 한 번 더 버틸 수 있습니다.', '자동 적용되며 빨간 하트가 1개 증가합니다.'],
    life2: ['생명 +2', '어려운 레벨을 위해 생명이 2개 증가합니다.', '자동 적용되며 빨간 하트가 2개 증가합니다.'],
    shorten: ['몸길이 -3', '뱀의 몸이 3칸 짧아집니다.', '자동 적용되어 회피가 쉬워집니다.'],
    removeObstacle: ['장애물 3개 제거', '보라색 장애물 3개를 제거합니다.', '자동 적용됩니다.'],
    clearBomb: ['폭탄 모두 제거', '파란 폭탄을 모두 제거합니다.', '자동 적용됩니다.'],
    slow: ['감속 10초', '10초 동안 뱀이 느려집니다.', '그동안 음식을 먹거나 안전한 길을 찾으세요.'],
    shield: ['보호막 10초', '10초 동안 장애물과 폭탄을 부술 수 있습니다.', '머리가 금색이면 보호막이 활성화된 상태입니다.'],
    hideObstacle: ['장애물 무효 10초', '10초 동안 보라색 장애물이 무효가 됩니다.', '그동안 안전하게 지나갈 수 있습니다.'],
  },
  es: {
    life1: ['Vida +1', 'Ganas 1 vida extra.', 'Se activa automáticamente y agrega un corazón rojo.'],
    life2: ['Vida +2', 'Ganas 2 vidas extra.', 'Se activa automáticamente y agrega dos corazones.'],
    shorten: ['Serpiente -3', 'La serpiente se acorta 3 bloques.', 'Se activa automáticamente.'],
    removeObstacle: ['Quitar 3 obstáculos', 'Elimina 3 obstáculos morados.', 'Se activa automáticamente.'],
    clearBomb: ['Eliminar bombas', 'Elimina todas las bombas azules.', 'Se activa automáticamente.'],
    slow: ['Lento 10s', 'La serpiente va más lenta por 10 segundos.', 'Usa el tiempo para buscar una ruta segura.'],
    shield: ['Escudo 10s', 'Por 10 segundos destruyes obstáculos y bombas.', 'Funciona automáticamente cuando la cabeza es dorada.'],
    hideObstacle: ['Obstáculos off 10s', 'Los obstáculos morados se desactivan por 10 segundos.', 'Puedes pasar por ellos mientras dura.'],
  },
  fr: {
    life1: ['Vie +1', 'Tu gagnes 1 vie supplémentaire.', 'Automatique, un cœur rouge est ajouté.'],
    life2: ['Vie +2', 'Tu gagnes 2 vies supplémentaires.', 'Automatique, deux cœurs sont ajoutés.'],
    shorten: ['Serpent -3', 'Le serpent raccourcit de 3 cases.', 'Automatique.'],
    removeObstacle: ['Retirer 3 obstacles', 'Supprime 3 obstacles violets.', 'Automatique.'],
    clearBomb: ['Supprimer les bombes', 'Supprime toutes les bombes bleues.', 'Automatique.'],
    slow: ['Ralentir 10s', 'Le serpent ralentit pendant 10 secondes.', 'Profite-en pour trouver une route sûre.'],
    shield: ['Bouclier 10s', 'Pendant 10 secondes, tu détruis obstacles et bombes.', 'Actif quand la tête est dorée.'],
    hideObstacle: ['Obstacles off 10s', 'Les obstacles violets sont désactivés 10 secondes.', 'Tu peux les traverser.'],
  },
  de: {
    life1: ['Leben +1', 'Du erhältst 1 zusätzliches Leben.', 'Automatisch, ein rotes Herz wird hinzugefügt.'],
    life2: ['Leben +2', 'Du erhältst 2 zusätzliche Leben.', 'Automatisch, zwei Herzen werden hinzugefügt.'],
    shorten: ['Schlange -3', 'Die Schlange wird 3 Felder kürzer.', 'Automatisch.'],
    removeObstacle: ['3 Hindernisse entfernen', 'Entfernt 3 lila Hindernisse.', 'Automatisch.'],
    clearBomb: ['Alle Bomben löschen', 'Entfernt alle blauen Bomben.', 'Automatisch.'],
    slow: ['Langsam 10s', 'Die Schlange wird 10 Sekunden langsamer.', 'Nutze die Zeit für sichere Wege.'],
    shield: ['Schild 10s', '10 Sekunden zerstörst du Hindernisse und Bomben.', 'Aktiv, wenn der Kopf golden ist.'],
    hideObstacle: ['Hindernisse aus 10s', 'Lila Hindernisse sind 10 Sekunden deaktiviert.', 'Du kannst hindurchgehen.'],
  },
  pt: {
    life1: ['Vida +1', 'Você ganha 1 vida extra.', 'Ativa automaticamente e adiciona um coração.'],
    life2: ['Vida +2', 'Você ganha 2 vidas extras.', 'Ativa automaticamente e adiciona dois corações.'],
    shorten: ['Cobra -3', 'A cobra fica 3 blocos menor.', 'Ativa automaticamente.'],
    removeObstacle: ['Remover 3 obstáculos', 'Remove 3 obstáculos roxos.', 'Ativa automaticamente.'],
    clearBomb: ['Limpar bombas', 'Remove todas as bombas azuis.', 'Ativa automaticamente.'],
    slow: ['Lento 10s', 'A cobra fica mais lenta por 10 segundos.', 'Use esse tempo para achar uma rota segura.'],
    shield: ['Escudo 10s', 'Por 10 segundos você destrói obstáculos e bombas.', 'Ativo quando a cabeça fica dourada.'],
    hideObstacle: ['Obstáculos off 10s', 'Obstáculos roxos ficam desativados por 10 segundos.', 'Você pode passar por eles.'],
  },
  ru: {
    life1: ['Жизнь +1', 'Добавляет 1 жизнь.', 'Срабатывает автоматически, появляется сердце.'],
    life2: ['Жизнь +2', 'Добавляет 2 жизни.', 'Срабатывает автоматически, появляются два сердца.'],
    shorten: ['Змейка -3', 'Змейка становится короче на 3 клетки.', 'Срабатывает автоматически.'],
    removeObstacle: ['Убрать 3 препятствия', 'Удаляет 3 фиолетовых препятствия.', 'Срабатывает автоматически.'],
    clearBomb: ['Убрать бомбы', 'Удаляет все синие бомбы.', 'Срабатывает автоматически.'],
    slow: ['Замедление 10с', 'Змейка замедляется на 10 секунд.', 'Используй время для безопасного пути.'],
    shield: ['Щит 10с', '10 секунд препятствия и бомбы уничтожаются.', 'Активен, когда голова золотая.'],
    hideObstacle: ['Препятствия выкл. 10с', 'Фиолетовые препятствия отключены на 10 секунд.', 'Можно проходить сквозь них.'],
  },
  ar: {
    life1: ['حياة +1', 'تحصل على حياة إضافية.', 'تعمل تلقائياً ويضاف قلب أحمر.'],
    life2: ['حياة +2', 'تحصل على حياتين إضافيتين.', 'تعمل تلقائياً ويضاف قلبان.'],
    shorten: ['تقليل الجسم -3', 'يصبح الثعبان أقصر بثلاث خانات.', 'تعمل تلقائياً.'],
    removeObstacle: ['إزالة 3 عوائق', 'تزيل 3 عوائق بنفسجية.', 'تعمل تلقائياً.'],
    clearBomb: ['إزالة القنابل', 'تزيل كل القنابل الزرقاء.', 'تعمل تلقائياً.'],
    slow: ['إبطاء 10 ثوانٍ', 'يصبح الثعبان أبطأ لمدة 10 ثوانٍ.', 'استغل الوقت لإيجاد طريق آمن.'],
    shield: ['درع 10 ثوانٍ', 'لمدة 10 ثوانٍ يتم تدمير العوائق والقنابل.', 'يعمل عندما يصبح الرأس ذهبياً.'],
    hideObstacle: ['تعطيل العوائق 10 ثوانٍ', 'العوائق البنفسجية تتعطل لمدة 10 ثوانٍ.', 'يمكن المرور خلالها.'],
  },
  hi: {
    life1: ['जीवन +1', 'आपको 1 अतिरिक्त जीवन मिलता है।', 'अपने आप लागू होता है और एक लाल दिल जुड़ता है।'],
    life2: ['जीवन +2', 'आपको 2 अतिरिक्त जीवन मिलते हैं।', 'अपने आप लागू होता है और दो दिल जुड़ते हैं।'],
    shorten: ['साँप -3', 'साँप 3 ब्लॉक छोटा हो जाता है।', 'अपने आप लागू होता है।'],
    removeObstacle: ['3 बाधाएँ हटाएँ', '3 बैंगनी बाधाएँ हटती हैं।', 'अपने आप लागू होता है।'],
    clearBomb: ['सभी बम हटाएँ', 'सभी नीले बम हटते हैं।', 'अपने आप लागू होता है।'],
    slow: ['धीमा 10 सेकंड', 'साँप 10 सेकंड धीमा होता है।', 'सुरक्षित रास्ता खोजने के लिए समय का उपयोग करें।'],
    shield: ['शील्ड 10 सेकंड', '10 सेकंड तक बाधाएँ और बम टूट जाते हैं।', 'सिर सुनहरा हो तो शील्ड सक्रिय है।'],
    hideObstacle: ['बाधा बंद 10 सेकंड', 'बैंगनी बाधाएँ 10 सेकंड निष्क्रिय होती हैं।', 'आप उनके बीच से गुजर सकते हैं।'],
  },
  id: {
    life1: ['Nyawa +1', 'Mendapat 1 nyawa tambahan.', 'Aktif otomatis dan menambah satu hati merah.'],
    life2: ['Nyawa +2', 'Mendapat 2 nyawa tambahan.', 'Aktif otomatis dan menambah dua hati merah.'],
    shorten: ['Ular -3', 'Tubuh ular berkurang 3 blok.', 'Aktif otomatis.'],
    removeObstacle: ['Hapus 3 rintangan', 'Menghapus 3 rintangan ungu.', 'Aktif otomatis.'],
    clearBomb: ['Hapus semua bom', 'Menghapus semua bom biru.', 'Aktif otomatis.'],
    slow: ['Lambat 10d', 'Ular melambat selama 10 detik.', 'Gunakan waktu ini untuk mencari jalur aman.'],
    shield: ['Perisai 10d', 'Selama 10 detik, rintangan dan bom dihancurkan.', 'Aktif saat kepala berwarna emas.'],
    hideObstacle: ['Rintangan mati 10d', 'Rintangan ungu tidak aktif selama 10 detik.', 'Kamu bisa melewatinya.'],
  },
  vi: {
    life1: ['Mạng +1', 'Bạn nhận thêm 1 mạng.', 'Tự động kích hoạt và thêm một tim đỏ.'],
    life2: ['Mạng +2', 'Bạn nhận thêm 2 mạng.', 'Tự động kích hoạt và thêm hai tim đỏ.'],
    shorten: ['Rắn -3', 'Thân rắn ngắn đi 3 ô.', 'Tự động kích hoạt.'],
    removeObstacle: ['Xóa 3 vật cản', 'Xóa 3 vật cản màu tím.', 'Tự động kích hoạt.'],
    clearBomb: ['Xóa tất cả bom', 'Xóa tất cả bom màu xanh.', 'Tự động kích hoạt.'],
    slow: ['Chậm 10 giây', 'Rắn chậm lại trong 10 giây.', 'Dùng thời gian này để tìm đường an toàn.'],
    shield: ['Khiên 10 giây', 'Trong 10 giây, vật cản và bom sẽ bị phá hủy.', 'Đầu màu vàng nghĩa là khiên đang bật.'],
    hideObstacle: ['Tắt vật cản 10 giây', 'Vật cản tím mất tác dụng trong 10 giây.', 'Bạn có thể đi xuyên qua chúng.'],
  },
  th: {
    life1: ['ชีวิต +1', 'ได้ชีวิตเพิ่ม 1 ดวง', 'ทำงานอัตโนมัติและเพิ่มหัวใจสีแดง 1 ดวง'],
    life2: ['ชีวิต +2', 'ได้ชีวิตเพิ่ม 2 ดวง', 'ทำงานอัตโนมัติและเพิ่มหัวใจสีแดง 2 ดวง'],
    shorten: ['งูสั้นลง -3', 'ตัวงูสั้นลง 3 ช่อง', 'ทำงานอัตโนมัติ'],
    removeObstacle: ['ลบสิ่งกีดขวาง 3 อัน', 'ลบสิ่งกีดขวางสีม่วง 3 อัน', 'ทำงานอัตโนมัติ'],
    clearBomb: ['ลบระเบิดทั้งหมด', 'ลบระเบิดสีน้ำเงินทั้งหมด', 'ทำงานอัตโนมัติ'],
    slow: ['ช้าลง 10 วิ', 'งูช้าลง 10 วินาที', 'ใช้เวลานี้หาเส้นทางปลอดภัย'],
    shield: ['โล่ 10 วิ', '10 วินาที ชนสิ่งกีดขวางหรือระเบิดแล้วไม่ตาย', 'หัวสีทองแปลว่าโล่ทำงาน'],
    hideObstacle: ['ปิดสิ่งกีดขวาง 10 วิ', 'สิ่งกีดขวางสีม่วงใช้ไม่ได้ 10 วินาที', 'สามารถผ่านได้อย่างปลอดภัย'],
  },
} satisfies Record<Language, Record<RewardKey, [string, string, string]>>

const t = computed(() => text[language.value])
const r = computed(() => rewardTexts[language.value])

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

const nextRewardPoint = computed(() => Math.floor(score.value / 10) * 10 + 10)

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

const activeShield = computed(() => shieldLeft.value > 0)
const activeSlow = computed(() => slowLeft.value > 0)
const obstaclesHidden = computed(() => hideObstacleLeft.value > 0)
const isInvincible = computed(() => invincibleLeft.value > 0)

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

function isObstacle(cell: Position) {
  return !obstaclesHidden.value && obstacles.value.some((item) => same(item, cell))
}

function isBomb(cell: Position) {
  return bombs.value.some((item) => same(item, cell))
}

function getCellClass(cell: Position) {
  if (isSnakeHead(cell)) {
    if (activeShield.value) return 'cell snake-head shield'
    if (isInvincible.value) return 'cell snake-head invincible'
    return 'cell snake-head'
  }

  if (isSnakeBody(cell)) return 'cell snake-body'
  if (isFood(cell)) return 'cell food'
  if (isBomb(cell)) return 'cell bomb'
  if (isObstacle(cell)) return 'cell obstacle'
  return 'cell'
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

function addSafeHazard(kind: 'obstacle' | 'bomb') {
  for (let i = 0; i < 400; i++) {
    const candidate = randomPosition()

    if (occupied(candidate)) continue
    if (!farFromSnakeHead(candidate)) continue
    if (Math.abs(candidate.x - food.value.x) + Math.abs(candidate.y - food.value.y) <= 1) continue

    if (kind === 'obstacle') {
      obstacles.value.push(candidate)
    } else {
      bombs.value.push(candidate)
    }

    const stillHasPath = pathExists(snake.value[0]!, food.value)
    const headHasSpace = freeNeighborCount(snake.value[0]!) >= 2
    const foodHasSpace = freeNeighborCount(food.value) >= 1

    if (stillHasPath && headHasSpace && foodHasSpace) {
      return
    }

    if (kind === 'obstacle') {
      obstacles.value.pop()
    } else {
      bombs.value.pop()
    }
  }
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
}

function clearHazardsOnSnake() {
  obstacles.value = obstacles.value.filter((item) => {
    return !snake.value.some((part) => same(part, item))
  })

  bombs.value = bombs.value.filter((item) => {
    return !snake.value.some((part) => same(part, item))
  })
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

function resetGameData() {
  snake.value = [{ x: 10, y: 10 }]
  food.value = { x: 5, y: 5 }
  obstacles.value = []
  bombs.value = []

  direction.value = { x: 1, y: 0 }
  nextDirection.value = { x: 1, y: 0 }

  score.value = 0
  lives.value = 1
  level.value = 1
  speed.value = baseSpeed

  lastDifficultyScore.value = 0
  lastRewardScore.value = 0

  currentSkillKey.value = null
  rewardKey.value = null

  shieldUntil.value = 0
  slowUntil.value = 0
  hideObstacleUntil.value = 0
  invincibleUntil.value = 0
  wasSlowActive = false

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
}

function handleCollision(target: Position, type: 'wall' | 'self' | 'obstacle' | 'bomb') {
  if (isInvincible.value) {
    return false
  }

  if ((type === 'obstacle' || type === 'bomb') && activeShield.value) {
    removeHazardAt(target)
    return false
  }

  if (type === 'bomb') {
    bombs.value = bombs.value.filter((item) => !same(item, target))
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
  if (score.value === lastDifficultyScore.value) return

  lastDifficultyScore.value = score.value
  level.value = difficultyStep + 1

  speed.value = Math.max(58, baseSpeed - difficultyStep * 10)

  addSafeHazard('obstacle')

  if (difficultyStep >= 2) {
    addSafeHazard('obstacle')
  }

  if (score.value >= 10) {
    addSafeHazard('bomb')
  }

  if (score.value >= 20 && difficultyStep % 2 === 0) {
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
  if (score.value === lastRewardScore.value) return

  lastRewardScore.value = score.value
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

function claimReward() {
  status.value = 'playing'
  startTimer()
}

function afterEatingFood() {
  score.value++
  createFood()

  if (score.value % 5 === 0) {
    increaseDifficulty()
  }

  if (score.value % 10 === 0) {
    triggerReward()
  }
}

function moveSnake() {
  if (status.value !== 'playing') return

  direction.value = nextDirection.value

  const head = snake.value[0]!
  const newHead = {
    x: head.x + direction.value.x,
    y: head.y + direction.value.y,
  }

  if (!inBoard(newHead)) {
    handleCollision(newHead, 'wall')
    return
  }

  if (snake.value.some((part) => same(part, newHead))) {
    handleCollision(newHead, 'self')
    return
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

  if (same(newHead, food.value)) {
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

onMounted(() => {
  window.addEventListener('keydown', changeDirection, { passive: false })

  clockTimer = window.setInterval(() => {
    now.value = Date.now()

    if (wasSlowActive && !activeSlow.value && status.value === 'playing') {
      wasSlowActive = false
      startTimer()
    }
  }, 100)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', changeDirection)
  clearGameTimer()

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
          <div class="language-box">
            <span>{{ t.language }}</span>
            <select v-model="language">
              <option
                v-for="item in languageOptions"
                :key="item.value"
                :value="item.value"
              >
                {{ item.label }}
              </option>
            </select>
          </div>

          <div class="status-box">
            <span>{{ t.status }}</span>
            <strong>{{ statusText }}</strong>
          </div>
        </div>
      </header>

      <main class="game-layout">
        <div class="left-area">
          <div class="board-frame">
            <div class="board-title">
              <span>{{ t.boardTitle }}</span>

              <div class="life-bar">
                <span class="life-label">{{ t.lives }}</span>
                <span
                  v-for="heart in lives"
                  :key="heart"
                  class="heart"
                >
                  ❤️
                </span>
              </div>

              <span v-if="activeShield" class="skill-badge">{{ r.shield[0] }} {{ shieldLeft }}{{ t.second }}</span>
              <span v-else-if="activeSlow" class="skill-badge">{{ r.slow[0] }} {{ slowLeft }}{{ t.second }}</span>
              <span v-else-if="obstaclesHidden" class="skill-badge">{{ r.hideObstacle[0] }} {{ hideObstacleLeft }}{{ t.second }}</span>
              <span v-else class="live-dot"></span>
            </div>

            <div class="board">
              <div
                v-for="(cell, index) in cells"
                :key="index"
                :class="getCellClass(cell)"
              ></div>
            </div>
          </div>

          <div class="bottom-dashboard">
            <div class="effect-panel">
              <h3>{{ t.activeEffects }}</h3>

              <div v-if="activeShield || activeSlow || obstaclesHidden" class="effect-list">
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
              </div>

              <p v-else class="no-effect">{{ t.noActiveEffect }}</p>
            </div>

            <div class="legend-panel">
              <h3>{{ t.legend }}</h3>

              <div class="legend-list">
                <div class="legend-item">
                  <span class="legend-dot food-dot"></span>
                  {{ t.legendFood }}
                </div>

                <div class="legend-item">
                  <span class="legend-dot obstacle-dot"></span>
                  {{ t.legendObstacle }}
                </div>

                <div class="legend-item">
                  <span class="legend-dot bomb-dot"></span>
                  {{ t.legendBomb }}
                </div>
              </div>
            </div>

            <div class="rules-panel">
              <h3>{{ t.rules }}</h3>
              <p>{{ t.ruleFood }}</p>
              <p>{{ t.ruleHazard }}</p>
              <p>{{ t.ruleLevel }}</p>
              <p>{{ t.ruleReward }}</p>

              <div class="skill-list-title">{{ t.allSkills }}</div>
              <div class="skill-list">
                <span
                  v-for="key in rewardKeys"
                  :key="key"
                  class="skill-pill"
                >
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

          <div class="current-skill-box">
            <p>{{ t.skill }}</p>
            <strong>{{ currentSkillName }}</strong>
          </div>

          <div class="info-box">
            <p>{{ t.speed }}：{{ snakeSpeedSeconds }} {{ t.secondPerStep }}</p>
            <p>{{ t.obstacles }}：{{ obstacles.length }}</p>
            <p>{{ t.bombs }}：{{ bombs.length }}</p>
          </div>

          <div class="button-group">
            <button
              v-if="status === 'ready' || status === 'over'"
              class="btn start"
              @click="startGame"
            >
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
  padding: 18px;
  color: white;
  font-family:
    Arial,
    "Microsoft JhengHei",
    sans-serif;
  background:
    radial-gradient(circle at top left, #7c3aed 0, transparent 28%),
    radial-gradient(circle at bottom right, #06b6d4 0, transparent 28%),
    linear-gradient(135deg, #111827, #1e1b4b, #0f172a);
}

.background-circle {
  position: absolute;
  border-radius: 50%;
  filter: blur(4px);
  opacity: 0.45;
  animation: float 5s infinite alternate ease-in-out;
}

.circle-one {
  width: 150px;
  height: 150px;
  left: 8%;
  top: 12%;
  background: #f97316;
}

.circle-two {
  width: 220px;
  height: 220px;
  right: 8%;
  bottom: 10%;
  background: #22c55e;
}

.circle-three {
  width: 110px;
  height: 110px;
  right: 20%;
  top: 14%;
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
  width: 1210px;
  padding: 20px;
  border: 1px solid rgba(255, 255, 255, 0.18);
  border-radius: 26px;
  background: rgba(15, 23, 42, 0.78);
  box-shadow: 0 30px 80px rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(18px);
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 22px;
  margin-bottom: 16px;
}

.tag {
  display: inline-block;
  margin: 0 0 6px;
  padding: 5px 11px;
  color: #67e8f9;
  background: rgba(8, 145, 178, 0.2);
  border: 1px solid rgba(103, 232, 249, 0.4);
  border-radius: 999px;
  font-size: 13px;
  letter-spacing: 1px;
}

h1 {
  margin: 0;
  font-size: 38px;
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
  gap: 12px;
  align-items: stretch;
}

.language-box {
  width: 205px;
  padding: 13px;
  border-radius: 18px;
  text-align: center;
  background: linear-gradient(135deg, #7c2d12, #be185d);
  box-shadow: 0 12px 30px rgba(244, 114, 182, 0.18);
}

.language-box span,
.status-box span {
  display: block;
  color: #ffe4e6;
  font-size: 13px;
  margin-bottom: 7px;
}

.language-box select {
  width: 100%;
  padding: 9px 10px;
  border: none;
  outline: none;
  border-radius: 12px;
  color: #0f172a;
  background: #fff7ed;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
}

.status-box {
  width: 140px;
  padding: 13px;
  border-radius: 18px;
  text-align: center;
  background: linear-gradient(135deg, #312e81, #0f766e);
  box-shadow: 0 12px 30px rgba(34, 211, 238, 0.18);
}

.status-box strong {
  font-size: 21px;
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
  background: linear-gradient(
    135deg,
    rgba(30, 41, 59, 0.9),
    rgba(15, 23, 42, 0.9)
  );
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
  border: 1px solid rgba(51, 65, 85, 0.7);
  background: rgba(15, 23, 42, 0.72);
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

@keyframes foodPulse {
  from {
    transform: scale(0.78);
  }

  to {
    transform: scale(1.05);
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

.legend-dot {
  width: 22px;
  height: 22px;
  display: inline-block;
  border-radius: 50%;
}

.food-dot {
  background: radial-gradient(circle, #ffffff, #fb7185, #e11d48);
  box-shadow: 0 0 12px #fb7185;
}

.obstacle-dot {
  border-radius: 7px;
  background: linear-gradient(135deg, #c084fc, #7e22ce, #3b0764);
  box-shadow: 0 0 10px rgba(168, 85, 247, 0.5);
}

.bomb-dot {
  background: radial-gradient(circle, #ffffff, #67e8f9, #0284c7, #020617);
  border: 2px solid #bae6fd;
  box-shadow: 0 0 12px #38bdf8;
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
  width: 330px;
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
.game-over-overlay {
  position: fixed;
  inset: 0;
  z-index: 20;
  display: grid;
  place-items: center;
  background: rgba(2, 6, 23, 0.72);
  backdrop-filter: blur(8px);
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
  }

  .language-box,
  .status-box {
    flex: 1;
    width: auto;
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
  .game-over-card {
    width: 92%;
  }
}
</style>