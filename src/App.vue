<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, nextTick } from "vue";
import { createClient } from "@supabase/supabase-js";

const SUPABASE_URL = "https://lhwpeveyqguonagheybd.supabase.co";
const SUPABASE_ANON = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Imxod3BldmV5cWd1b25hZ2hleWJkIiwicm9sZSI6ImFub24iLCJpYXQiOjE3Nzk5NzU5ODMsImV4cCI6MjA5NTU1MTk4M30.9iMXnjWLPSu-n1OFPsTcC3dtxrzZ2VJbSoX2DQErVwk";
const sb = createClient(SUPABASE_URL, SUPABASE_ANON);

const SIZE = 480;
const GRID = 20;
const CELL = SIZE / GRID;

// ── Hamiltonian Cycle（蛇行順序，覆蓋全地圖）──
const HC:(number[]|undefined)[]=[];  // HC[x][y] = cycle index (0~399)
const HC_INV:{x:number;y:number}[]=[]; // HC_INV[idx] = {x,y}
for(let x=0;x<GRID;x++){const r:number[]=[];for(let y=0;y<GRID;y++)r[y]=-1;HC[x]=r;}
let _idx=0;
for(let y=0;y<GRID;y++){
  if(y%2===0)for(let x=0;x<GRID;x++){(HC[x]as number[])[y]=_idx;HC_INV[_idx]={x,y};_idx++;}
  else for(let x=GRID-1;x>=0;x--){(HC[x]as number[])[y]=_idx;HC_INV[_idx]={x,y};_idx++;}
}

const state      = ref<"menu"|"playing"|"gameover">("menu");
const selectedMode = ref<"main"|"normal"|"tron"|"boss"|"stage"|"shop">("main");
const score      = ref(0);
const playerName = ref("");
const playerName2 = ref("");  // P2 名稱（鬼抓人）
const nameError  = ref(false);
const isAI       = ref(false);
const paused     = ref(false);
const selectedPU = ref<PUType|null>(null);
const is3D = ref(false);
const canvasWidth = computed(() => gameMode.value === "tron" ? SIZE * 2 : SIZE);
const canvasHeight = computed(() => gameMode.value === "tron" ? SIZE + 80 : SIZE);
const containerRef = ref<HTMLDivElement|null>(null);
const activePUsCopy = ref<ActivePU[]>([]);

const canvasRef = ref<HTMLCanvasElement|null>(null);
let ctx: CanvasRenderingContext2D|null = null;

let snake: {x:number;y:number}[] = [];
let dir: "UP"|"DOWN"|"LEFT"|"RIGHT" = "RIGHT";
const dirQueue: ("UP"|"DOWN"|"LEFT"|"RIGHT")[] = [];
let snake2: {x:number;y:number}[] = [];
let dir2: "UP"|"DOWN"|"LEFT"|"RIGHT" = "LEFT";
const dirQueue2: ("UP"|"DOWN"|"LEFT"|"RIGHT")[] = [];
let food = {x:5,y:5};

type PUType = "shield"|"ghost"|"double"|"shrink"|"slow"|"trap"|"remove-trap";
let fieldPU: {x:number;y:number;type:PUType;life:number}|null = null;

interface ActivePU { type:PUType; steps:number; maxSteps:number; }
let activePUs: ActivePU[] = [];

interface Particle { x:number;y:number;vx:number;vy:number;life:number;maxLife:number;color:string;size:number; }
let particles: Particle[] = [];

let lastFoodStep=0, currentStep=0, combo=0;
let comboText: {text:string;x:number;y:number;life:number}|null = null;
let level=1, levelFlash=0, shakeFrames=0, isDying=false;
let tick=0;

// ── color palettes ────────────────────────────────────────
type ColorKey = "green"|"blue"|"red"|"purple"|"orange"|"cyan";
interface ColorPalette { head:[number,number,number]; tail:[number,number,number]; }
const COLOR_PALETTES:Record<ColorKey,ColorPalette> = {
  green:{head:[0,255,200],tail:[0,80,60]},
  blue:{head:[0,150,255],tail:[0,50,100]},
  red:{head:[255,50,50],tail:[100,0,0]},
  purple:{head:[200,100,255],tail:[80,40,120]},
  orange:{head:[255,180,0],tail:[100,70,0]},
  cyan:{head:[0,255,255],tail:[0,100,100]},
};
const colorKeys:ColorKey[] = ["green","blue","red","purple","orange","cyan"];
const playerColor = ref<ColorKey>("green");
const aiColor = ref<ColorKey>("purple");

// ── held keys for continuous fire ─────────────────────────
const pressedKeys = new Set<string>();

// ── shop ──────────────────────────────────────────────────
interface ShopItem { id:string; name:string; desc:string; cost:number; icon:string; }
const SHOP_ITEMS:ShopItem[] = [
  {id:"startShield", name:"開局護盾", desc:"每場開始時自帶護盾（抵擋一次碰撞）", cost:50, icon:"🛡"},
  {id:"startSlow", name:"開局緩速", desc:"每場開始時自帶緩速效果", cost:30, icon:"⏱"},
  {id:"extraLife", name:"額外生命", desc:"每場有一次復活機會", cost:100, icon:"❤️"},
  {id:"startDouble", name:"開局雙倍", desc:"每場開始時自帶 20 步雙倍分數", cost:80, icon:"💎"},
];
const coins = ref(0);
const ownedItems = ref<Set<string>>(new Set());
function loadShop() {
  try{ coins.value=parseInt(localStorage.getItem("ns-coins")||"0",10); }catch{coins.value=0;}
  try{
    const r=localStorage.getItem("ns-shop");
    ownedItems.value=new Set(r?JSON.parse(r):[]);
  }catch{ownedItems.value=new Set();}
}
function saveShop() {
  try{ localStorage.setItem("ns-coins",String(coins.value)); }catch{}
  try{ localStorage.setItem("ns-shop",JSON.stringify([...ownedItems.value])); }catch{}
}
function buyItem(id:string) {
  const item=SHOP_ITEMS.find(i=>i.id===id);
  if(!item||ownedItems.value.has(id)||coins.value<item.cost) return;
  coins.value-=item.cost;
  ownedItems.value.add(id);
  saveShop();
}

// ── stage mode ────────────────────────────────────────────
const MAX_STAGE = 20;
const stage = ref(1);
const portal = ref<{x:number;y:number}|null>(null);
let traps:{x:number;y:number}[] = [];
function getStageTarget(s:number):number {
  return Math.min(3 + (s-1) * 3, 50);
}
function getStageSpeed(s:number):number {
  return Math.max(35, 130 - s * 5);
}

// ── game mode ────────────────────────────────────────────
const gameMode = ref<"normal"|"tron"|"boss"|"stage">("normal");

// ── Duel mode (雙人對戰) ───────────────────────────────────
const tronWinner = ref("");
const is2PAI = ref(false);
const DUEL_CW = 24;           // cell size (480/20)
const DUEL_CH = 24;           // same as CW now (square cells)
const DUEL_GW = 20 * DUEL_CW; // 480 — grid width
const DUEL_GH = 20 * DUEL_CH; // 480 — grid height
const OPPONENT_STEPS = 150;       // steps in opponent grid
let lives1 = 3, lives2 = 3;
let score1 = 0, score2 = 0;
let food2 = {x:15,y:5};
let traps1:{x:number;y:number}[] = [];
let traps2:{x:number;y:number}[] = [];
let portal1:{x:number;y:number}|null = null;
let portal2:{x:number;y:number}|null = null;
let portalCD1 = 0, portalCD2 = 0;
const AI_TRASH = [
  "新手村還沒畢業嗎？",
  "EZ.....",
  "GG",
  "你不會急眼了吧？",
  "菜就多練",
  "急了急了真的急了！",
  "嘻嘻",
];
let inOpponentGrid1 = false, inOpponentGrid2 = false;
let opponentTimer1 = 0, opponentTimer2 = 0;
let heldSkill1:PUType|null = null, heldSkill2:PUType|null = null;

// ── Boss Battle ──────────────────────────────────────────
interface BossData {
  body: {x:number;y:number}[];
  dir: "UP"|"DOWN"|"LEFT"|"RIGHT";
  health: number; maxHealth: number;
  active: boolean; alert: number;
}
let bossData: BossData|null = null;
interface MissileData { x:number; y:number; dx:number; dy:number; life:number; }
let missiles: MissileData[] = [];
const BOSS_THRESHOLD = 15;
const bossKills = ref(0);

// ── per-mode leaderboard (Supabase) & history (localStorage) ─
type LBEntry = {name:string;score:number;created_at?:string};
type Frame = {snake:{x:number;y:number}[];food:{x:number;y:number};fieldPU:{x:number;y:number;type:PUType;life:number}|null;activePUs:ActivePU[];score:number;bossData:BossData|null;missiles:MissileData[];portal?:{x:number;y:number}|null;traps?:{x:number;y:number}[];stage?:number;};
type HistoryItem = {label:string;score:number;frames:Frame[];mode?:string};
const leaderboard = ref<LBEntry[]>([]);
const history = ref<HistoryItem[]>([]);
let frames:Frame[] = [];
let replayTimer:ReturnType<typeof setTimeout>|null = null;

async function loadLB(mode:string) {
  leaderboard.value = [];
  try {
    const { data, error } = await sb
      .from("leaderboard")
      .select("name,score")
      .eq("mode", mode)
      .order("score", { ascending: false })
      .limit(10);
    if(error) { console.error("loadLB error", error); return; }
    if(data) leaderboard.value = data;
  } catch(e) { console.error("loadLB catch", e); }
}
async function addLB(mode:string, name:string, s:number) {
  try {
    const { error } = await sb.from("leaderboard").insert({ name, score: s, mode });
    if(error) console.error("addLB error", error);
  } catch(e) { console.error("addLB catch", e); }
  await loadLB(mode);
}
function loadHist(mode:string) {
  try {
    const raw = localStorage.getItem(`ns-hist-${mode}`);
    history.value = raw ? JSON.parse(raw) : [];
  } catch { history.value = []; }
}
function saveHist(mode:string) {
  try { localStorage.setItem(`ns-hist-${mode}`, JSON.stringify(history.value)); } catch {}
}
async function selectMode(mode:"main"|"normal"|"tron"|"boss"|"stage"|"shop") {
  selectedMode.value = mode;
  if(mode!=="main"){ await loadLB(mode); loadHist(mode); }
}

// ═══════════════════════════════════════════════════════════════
// ── AI 學習系統（從失敗教訓 + 真人高手學習）──
// ═══════════════════════════════════════════════════════════════
const AI_LEARN_KEY="ns-ai-learn-v2";
const HUMAN_TRAJ_KEY="ns-human-traj";

interface AILearnEntry{deaths:number;penalty:number;}
let aiLearn:Record<string,AILearnEntry>={};
let humanTraj:{headIdx:number;dir:string;snakeLen:number}[]=[];

function loadAILearn(){
  try{const raw=localStorage.getItem(AI_LEARN_KEY);if(raw)aiLearn=JSON.parse(raw);}catch{aiLearn={};}
  fetchGlobalLearn(); // sync from server
}
function saveAILearn(){
  try{localStorage.setItem(AI_LEARN_KEY,JSON.stringify(aiLearn));}catch{}
}

async function fetchGlobalLearn(){
  try{
    const {data}=await sb.from("ai_learning").select("data").limit(1).single();
    if(data?.data)for(const[k,v]of Object.entries(data.data as Record<string,AILearnEntry>)){
      if(!aiLearn[k]||v.penalty>aiLearn[k].penalty)aiLearn[k]=v;
    }
  }catch{}
}
async function syncAILearn(){
  try{
    const{data}=await sb.from("ai_learning").select("id").limit(1).maybeSingle();
    if(data?.id)await sb.from("ai_learning").update({data:aiLearn}).eq("id",(data as any).id);
    else await sb.from("ai_learning").insert({data:aiLearn});
  }catch{}
}
let _syncTimer=0;
function debounceSync(){clearTimeout(_syncTimer);_syncTimer=setTimeout(syncAILearn,3000)as any;}

// AI 死亡學習：記錄 (HC位置, 方向) → penalty
function recordAIDeath(headIdx:number,dir:string,snakeLen:number,space:number){
  const key=`${headIdx}:${dir}`;
  if(!aiLearn[key])aiLearn[key]={deaths:0,penalty:0};
  aiLearn[key].deaths++;
  aiLearn[key].penalty+=Math.max(1,300-space*(snakeLen>50?0.5:1));
  saveAILearn();debounceSync();
}

// 查詢 penalty：這個位置+方向以前死過幾次
function getLearnPenalty(headIdx:number,dir:string,snakeLen:number):number{
  const e=aiLearn[`${headIdx}:${dir}`];if(!e)return 0;
  const ls=snakeLen>80?2:snakeLen>50?1.5:1;
  return Math.min(e.penalty*ls,200000);
}
// 查詢真人 bonus：這個位置+方向真人常走嗎
function getHumanBonus(headIdx:number,dir:string,snakeLen:number):number{
  const matches=humanTraj.filter(m=>m.headIdx===headIdx&&m.dir===dir&&Math.abs(m.snakeLen-snakeLen)<15);
  return Math.min(matches.length*5000,100000);
}

// 真人軌跡學習
function recordHumanMove(curHeadIdx:number,dir:string,snakeLen:number){
  humanTraj.push({headIdx:curHeadIdx,dir,snakeLen});
}
function saveHumanTrajectory(){
  if(score.value<30)return;
  try{
    const ex=JSON.parse(localStorage.getItem(HUMAN_TRAJ_KEY)||"[]");
    ex.push({traj:humanTraj,score:score.value,date:Date.now(),mode:gameMode.value});
    ex.sort((a:any,b:any)=>b.score-a.score);
    if(ex.length>15)ex.length=15;
    localStorage.setItem(HUMAN_TRAJ_KEY,JSON.stringify(ex));
  }catch{}
}
function loadHumanTrajectories(){
  try{
    const raw=localStorage.getItem(HUMAN_TRAJ_KEY);
    if(raw){const all=JSON.parse(raw);humanTraj=[];for(const s of all)if(s.score>=30)humanTraj.push(...s.traj);}
  }catch{humanTraj=[];}
}

loadAILearn();
loadHumanTrajectories();

function recordFrame() {
  frames.push({
    snake:JSON.parse(JSON.stringify(snake)),
    food:{...food},
    fieldPU:fieldPU?{...fieldPU}:null,
    activePUs:JSON.parse(JSON.stringify(activePUs)),
    score:score.value,
    bossData:bossData?JSON.parse(JSON.stringify(bossData)):null,
    missiles:JSON.parse(JSON.stringify(missiles)),
    portal:portal.value?{...portal.value}:null,
    traps:JSON.parse(JSON.stringify(traps)),
    stage:stage.value,
  });
}

let last=0, acc=0, running=false;
function interval() {
  const hasSlow = activePUs.some(p=>p.type==="slow");
  let base = Math.max(55, 130-(level-1)*8);
  if(gameMode.value==="stage") base = getStageSpeed(stage.value);
  return hasSlow ? base*1.7 : base;
}

// ── power-up config ────────────────────────────────────────
const PU_COLOR:Record<PUType,string> = {shield:"#60a5fa",ghost:"#c084fc",double:"#fbd74b",shrink:"#f87171",slow:"#34d399",trap:"#f87171","remove-trap":"#2dd4bf"};
const PU_ICON:Record<PUType,string>  = {shield:"🛡",ghost:"👻",double:"💎",shrink:"✂️",slow:"⏱",trap:"🗑","remove-trap":"🧹"};
const PU_LABEL:Record<PUType,string> = {shield:"SHIELD",ghost:"GHOST",double:"2×SCORE",shrink:"SHRINK",slow:"SLOW-MO",trap:"TRAP","remove-trap":"CLEANSE"};
const PU_DESC:Record<PUType,string>  = {
  shield:"抵擋一次碰撞（牆壁或自身），激活後立即消耗。吃到後可放心撞一次！",
  ghost:"穿越牆壁，從另一側出現，持續約 32 步。讓蛇自由穿梭地圖！",
  double:"所有得分翻倍，持續約 40 步。連 Combo 也會跟著加成！",
  shrink:"蛇身立即縮短一半，幫你脫離死局。危急時的逃生利器！",
  slow:"移動速度減慢約 1.7×，持續約 38 步。給自己更多反應時間！",
  trap:"在對手場地放置一個陷阱！陷阱永久存在，踩到即死，需用清除技能移除。",
  "remove-trap":"清除自己場地上的所有陷阱！對手的陷阱瞬間消失無蹤！",
};
const PU_STEPS:Record<PUType,number> = {shield:10,ghost:32,double:40,shrink:0,slow:38,trap:0,"remove-trap":0};
const PU_TYPES:PUType[] = ["shield","ghost","double","shrink","slow"];

const puColor = (t:PUType) => PU_COLOR[t];
const puIcon  = (t:PUType) => PU_ICON[t];

function togglePUInfo(t: PUType) {
  selectedPU.value = selectedPU.value === t ? null : t;
}

// ── mode-specific items ─────────────────────────────────────
const MODE_ITEMS:Record<string,{id:string;icon:string;label:string;desc:string}[]> = {
  boss:[
    {id:"boss-missile",icon:"🚀",label:"飛彈",desc:"場上會出現飛彈道具，撿到後按 [SPACE] 發射，可對 Boss 造成傷害！"},
    {id:"boss-boss",icon:"👑",label:"Boss 戰",desc:"分數達到 15 分時 Boss 蛇現身追殺你，擊敗可獲得大量加分獎勵！"},
  ],
  stage:[
    {id:"stage-portal",icon:"🕳️",label:"黑洞傳送門",desc:"達到目標分數後出現紫色黑洞，蛇頭鑽進去就晉級下一關，並獲得金幣！"},
    {id:"stage-trap",icon:"🗑",label:"陷阱",desc:"後期關卡場上會出現紅色陷阱擋路，碰到會扣一條命！"},
  ],
  tron:[
    {id:"tron-portal",icon:"🌀",label:"傳送門",desc:"穿越傳送門可入侵對手場地，在對手地盤吃食物、放陷阱！"},
    {id:"tron-life",icon:"❤️",label:"生命系統",desc:"每人 3 條命，擊殺對手可偷取一條命，率先耗盡對方生命者獲勝！"},
    {id:"tron-trap",icon:"🗑",label:"陷阱技能",desc:"吃到 TRAP 技能道具後按空白鍵 (P1) 或 0 鍵 (P2)，在對手場地放一個永久陷阱！踩到即死，需用 🧹 CLEANSE 清除！"},
    {id:"tron-cleanse",icon:"🧹",label:"清除陷阱",desc:"場上有機會出現 🧹 CLEANSE 技能道具，撿到後按空白鍵 (P1) 或 0 鍵 (P2)，清除自己場地上所有陷阱！"},
  ],
};
const selectedModeItem = ref<string|null>(null);
function toggleModeItem(id:string){ selectedModeItem.value=selectedModeItem.value===id?null:id; }
function modeItem(mode:string){ return MODE_ITEMS[mode]?.find(i=>i.id===selectedModeItem.value)??null; }
function togglePause() {
  if (state.value !== "playing") return;
  paused.value = !paused.value;
}

function resumeGame() {
  paused.value = false;
}

function goMenuFromPause() {
  paused.value = false;
  state.value = "menu";
}

// ── utils ──────────────────────────────────────────────────
function randPos(excl:{x:number;y:number}[]=[]) {
  let p;
  do { p={x:Math.floor(Math.random()*GRID),y:Math.floor(Math.random()*GRID)}; }
  while(excl.some(e=>e.x===p!.x&&e.y===p!.y));
  return p;
}
function spawnFood() { food=randPos([...snake,...(fieldPU?[fieldPU]:[]),...(gameMode.value==="stage"?traps:[])]);  }
function trySpawnPU() {
  if(fieldPU) return;
  if(gameMode.value==="stage" && portal.value) return;
  if(Math.random()<0.28) {
    const excl = [snake,food] as any;
    if(gameMode.value==="stage") excl.push(...traps);
    fieldPU={...randPos([...snake,food,...(gameMode.value==="stage"?traps:[])]),type:PU_TYPES[Math.floor(Math.random()*PU_TYPES.length)]!,life:22};
  }
}
function burst(cx:number,cy:number,color:string,n=14) {
  for(let i=0;i<n;i++){
    const a=(Math.PI*2*i/n)+Math.random()*.5, sp=1.5+Math.random()*3.5;
    particles.push({x:cx,y:cy,vx:Math.cos(a)*sp,vy:Math.sin(a)*sp,life:30+Math.random()*25|0,maxLife:55,color,size:2+Math.random()*3});
  }
}
function lerp(a:number,b:number,t:number){return a+(b-a)*Math.min(1,Math.max(0,t));}

// ── canvas ─────────────────────────────────────────────────
async function initCanvas() {
  await nextTick();
  const c=canvasRef.value; if(!c) return false;
  ctx=c.getContext("2d"); return!!ctx;
}

// ── start ──────────────────────────────────────────────────
async function startGame(mode:"normal"|"tron"|"boss"|"stage", ai:boolean) {
  if(mode!=="tron"&&!ai&&!playerName.value.trim()){nameError.value=true;return;}
  nameError.value=false;
  gameMode.value=mode; isAI.value=ai; is2PAI.value=false;

  // ── 先初始化所有蛇（mode 專屬的在 state 變更前做完）──
  if(mode==="tron"){
    snake=[{x:3,y:10},{x:2,y:10},{x:1,y:10}];
    dir="RIGHT";
    snake2=[{x:16,y:10},{x:17,y:10},{x:18,y:10}];
    dir2="LEFT";
    if(ai) is2PAI.value=true;
    lives1=3;
    lives2=3;
    score1=0; score2=0;
    food2={x:15,y:5};
    traps1=[]; traps2=[]; portal1=null; portal2=null;
    portalCD1=0; portalCD2=0;
    inOpponentGrid1=false; inOpponentGrid2=false;
    opponentTimer1=0; opponentTimer2=0;
    heldSkill1=null; heldSkill2=null;
  } else {
    snake=[{x:10,y:10},{x:9,y:10},{x:8,y:10}];
    dir="RIGHT";
    snake2=[];  // 非 Tron 模式清空蛇2
  }
  if(mode==="boss"){snake=[{x:10,y:10},{x:9,y:10},{x:8,y:10}];dir="RIGHT";}
  if(mode==="stage"){snake=[{x:10,y:10},{x:9,y:10},{x:8,y:10}];dir="RIGHT";stage.value=1;portal.value=null;traps=[];}

  if(ai){
    const alt=colorKeys.filter(k=>k!==playerColor.value);
    aiColor.value=alt.length>0?alt[0]!:"purple";
  }
  score.value=0; frames=[]; particles=[];
  activePUs=[]; activePUs2=[]; fieldPU=null; fieldPU2=null; combo=0; level=1;
  heldPU.value=null;
  currentStep=0; lastFoodStep=0; levelFlash=0; shakeFrames=0;
  isDying=false; comboText=null; paused.value=false; humanTraj=[];
  tronWinner.value=""; bossData=null; missiles=[]; bossKills.value=0;
  missileCooldown=0; pressedKeys.clear(); extraLifeUsed=false;

  // ── 套用商店物品（存入手持技能，按空白鍵施放） ──
  if(mode!=="tron"){
    if(ownedItems.value.has("startShield") && !heldPU.value) heldPU.value="shield";
    if(ownedItems.value.has("startSlow") && !heldPU.value) heldPU.value="slow";
    if(ownedItems.value.has("startDouble") && !heldPU.value) heldPU.value="double";
  }
  const extraLifeOwned = ownedItems.value.has("extraLife");

  state.value="playing";

  if(!(await initCanvas())) return;
  spawnFood(); combo=0;
  last=performance.now(); acc=0;
  if(!running){running=true;requestAnimationFrame(loop);}
}
let extraLifeUsed = false;
async function restart() { await startGame(gameMode.value as any, isAI.value); }
function goMenu() { paused.value=false; state.value="menu"; selectMode("main"); }
function goModeDetail(mode:"normal"|"tron"|"boss"|"stage") { state.value="menu"; selectMode(mode); }

// ── AI ─────────────────────────────────────────────────────
function aiStep() {
  if(!snake.length) return;
  const head=snake[0]!;
  const D:("UP"|"DOWN"|"LEFT"|"RIGHT")[]=["UP","DOWN","LEFT","RIGHT"];
  const OPP:Record<string,string>={UP:"DOWN",DOWN:"UP",LEFT:"RIGHT",RIGHT:"LEFT"};
  const dv:Record<string,{x:number;y:number}>={UP:{x:0,y:-1},DOWN:{x:0,y:1},LEFT:{x:-1,y:0},RIGHT:{x:1,y:0}};
  const ghost=activePUs.some(p=>p.type==="ghost");
  const tail=snake[snake.length-1]!;

  function bfs(s:{x:number;y:number},t:{x:number;y:number},obs:Set<string>):"UP"|"DOWN"|"LEFT"|"RIGHT"|null {
    const vis=new Set(obs); vis.add(`${s.x},${s.y}`);
    const q:{x:number;y:number;d:"UP"|"DOWN"|"LEFT"|"RIGHT"|null}[]=[{...s,d:null}];
    while(q.length){
      const {x,y,d}=q.shift()!;
      if(x===t.x&&y===t.y) return d;
      for(const dd of D){
        let nx=x+dv[dd]!.x,ny=y+dv[dd]!.y;
        if(ghost){nx=(nx+GRID)%GRID;ny=(ny+GRID)%GRID;}
        else if(nx<0||ny<0||nx>=GRID||ny>=GRID) continue;
        const k=`${nx},${ny}`;
        if(vis.has(k)) continue; vis.add(k); q.push({x:nx,y:ny,d:d??dd});
      }
    }
    return null;
  }

  function floodFill(sx:number,sy:number,obs:Set<string>):number {
    const vis=new Set(obs); vis.add(`${sx},${sy}`);
    const q:[number,number][]=[[sx,sy]]; let cnt=0;
    while(q.length){
      const [x,y]=q.shift()!; cnt++;
      for(const d of D){
        let nx=x+dv[d]!.x,ny=y+dv[d]!.y;
        if(ghost){nx=(nx+GRID)%GRID;ny=(ny+GRID)%GRID;}
        else if(nx<0||ny<0||nx>=GRID||ny>=GRID) continue;
        const k=`${nx},${ny}`;
        if(vis.has(k)) continue; vis.add(k); q.push([nx,ny]);
      }
    } return cnt;
  }

  // obstacles for body segments (without head at 0, optionally without tail at end)
  function bodyObs(includeTail:boolean):Set<string> {
    const s=new Set<string>();
    for(let i=1;i<snake.length-(includeTail?0:1);i++) s.add(`${snake[i]!.x},${snake[i]!.y}`);
    return s;
  }

  // collect valid moves
  const moves:{d:"UP"|"DOWN"|"LEFT"|"RIGHT";x:number;y:number}[]=[];
  for(const d of D){
    if(d===OPP[dir]) continue;
    let nx=head.x+dv[d]!.x,ny=head.y+dv[d]!.y;
    if(ghost){nx=(nx+GRID)%GRID;ny=(ny+GRID)%GRID;}
    else if(nx<0||ny<0||nx>=GRID||ny>=GRID) continue;
    if(!ghost&&snake.some(s=>s.x===nx&&s.y===ny)) continue;
    moves.push({d,x:nx,y:ny});
  }
  if(!moves.length) return;

  // body excluding tail (tail moves away, so it's not an obstacle for pathfinding)
  const bodyNoTail=bodyObs(false);

  // ── try to reach a target safely ──
  function tryTarget(tx:number,ty:number):"UP"|"DOWN"|"LEFT"|"RIGHT"|null {
    const d=bfs(head,{x:tx,y:ty},bodyNoTail);
    if(!d||d===OPP[dir]) return null;
    const nx=head.x+dv[d]!.x,ny=head.y+dv[d]!.y;
    const eating=nx===food.x&&ny===food.y;
    // projected obstacles after this move
    const obs=bodyObs(eating);
    obs.add(`${head.x},${head.y}`);
    if(!eating) obs.delete(`${tail.x},${tail.y}`);
    // must have enough room after the move
    if(floodFill(nx,ny,obs)<3) return null;
    // if eating, also verify we can still reach the tail after (snake grows by 1)
    if(eating&&snake.length>3){
      const tailObs=new Set(obs);
      tailObs.delete(`${tail.x},${tail.y}`); // tail is the target, not obstacle
      if(!bfs({x:nx,y:ny},tail,tailObs)) return null;
    }
    return d;
  }

  // Strategy 1: valuable PU
  if(fieldPU&&["shield","ghost","double"].includes(fieldPU.type)){
    const d=tryTarget(fieldPU.x,fieldPU.y);
    if(d){dir=d;return;}
  }

  // Strategy 2: food (if safe)
  const fd=tryTarget(food.x,food.y);
  if(fd){dir=fd;return;}

  // Strategy 3: chase tail (safest survival)
  const fullBody=bodyObs(true);
  fullBody.add(`${head.x},${head.y}`);
  const td=bfs(head,tail,fullBody);
  if(td&&td!==OPP[dir]&&moves.some(m=>m.d===td)){dir=td;return;}

  // Strategy 4: flood fill – pick max space
  let best=moves[0]!.d,bestSpace=-1;
  for(const m of moves){
    const eating=m.x===food.x&&m.y===food.y;
    const obs=bodyObs(eating);
    obs.add(`${head.x},${head.y}`);
    const space=floodFill(m.x,m.y,obs);
    if(space>bestSpace){bestSpace=space;best=m.d;}
  }
  dir=best;
}

// ── input ──────────────────────────────────────────────────
const OPPOSITE:Record<string,string>={UP:"DOWN",DOWN:"UP",LEFT:"RIGHT",RIGHT:"LEFT"};
function pushDir(q:("UP"|"DOWN"|"LEFT"|"RIGHT")[], d:"UP"|"DOWN"|"LEFT"|"RIGHT", lastDir:"UP"|"DOWN"|"LEFT"|"RIGHT") {
  if(d===lastDir||d===OPPOSITE[lastDir]) return;
  if(q.length<2) q.push(d);
}
function key(e:KeyboardEvent) {
  if(state.value==="playing" && (e.key==="Escape"||e.key==="p"||e.key==="P")) {
    e.preventDefault(); togglePause(); return;
  }
  if(state.value!=="playing"||paused.value) return;

  // Player 1: WASD or Arrow keys — 單人 AI 模式才鎖住
  if(!isAI.value||gameMode.value==="tron"){
    const p1Arrow = gameMode.value!=="tron"||is2PAI.value;
    if(e.key==="w"||e.key==="W"||(p1Arrow&&e.key==="ArrowUp"))   {e.preventDefault();pushDir(dirQueue,"UP",dirQueue.length?dirQueue[dirQueue.length-1]!:dir);}
    if(e.key==="s"||e.key==="S"||(p1Arrow&&e.key==="ArrowDown")) {e.preventDefault();pushDir(dirQueue,"DOWN",dirQueue.length?dirQueue[dirQueue.length-1]!:dir);}
    if(e.key==="a"||e.key==="A"||(p1Arrow&&e.key==="ArrowLeft")) {e.preventDefault();pushDir(dirQueue,"LEFT",dirQueue.length?dirQueue[dirQueue.length-1]!:dir);}
    if(e.key==="d"||e.key==="D"||(p1Arrow&&e.key==="ArrowRight")){e.preventDefault();pushDir(dirQueue,"RIGHT",dirQueue.length?dirQueue[dirQueue.length-1]!:dir);}
  }

  // Player 2: Arrow keys (Duel mode) — AI 模式時已鎖(!is2PAI)
  if(gameMode.value==="tron"&&!is2PAI.value){
    if(e.key==="ArrowUp")   {e.preventDefault();pushDir(dirQueue2,"UP",dirQueue2.length?dirQueue2[dirQueue2.length-1]!:dir2);}
    if(e.key==="ArrowDown") {e.preventDefault();pushDir(dirQueue2,"DOWN",dirQueue2.length?dirQueue2[dirQueue2.length-1]!:dir2);}
    if(e.key==="ArrowLeft") {e.preventDefault();pushDir(dirQueue2,"LEFT",dirQueue2.length?dirQueue2[dirQueue2.length-1]!:dir2);}
    if(e.key==="ArrowRight"){e.preventDefault();pushDir(dirQueue2,"RIGHT",dirQueue2.length?dirQueue2[dirQueue2.length-1]!:dir2);}
  }

  // Skill activation: SPACE
  if(e.key===" ") {
    if(gameMode.value==="tron"){
      e.preventDefault();
      if(heldSkill1) activateDuelSkill(1);
    } else if(gameMode.value==="boss"){
      e.preventDefault();
      pressedKeys.add(" ");
    } else {
      // 非對戰模式：使用商店購買的技能
      if(heldPU.value){
        e.preventDefault();
        const pu=heldPU.value; heldPU.value=null;
        // 消耗對應的商店道具
        const shopMap:Record<string,string>={shield:"startShield",slow:"startSlow",double:"startDouble"};
        const sid=shopMap[pu]; if(sid&&ownedItems.value.has(sid)){ownedItems.value.delete(sid);saveShop();}
        if(pu==="shrink"){
          snake=snake.slice(0,Math.max(1,snake.length>>1));
        } else {
          const ex=activePUs.find(p=>p.type===pu);
          if(ex) ex.steps=PU_STEPS[pu];
          else activePUs.push({type:pu,steps:PU_STEPS[pu],maxSteps:PU_STEPS[pu]});
        }
        burst(snake[0]!.x*CELL+CELL/2,snake[0]!.y*CELL+CELL/2,PU_COLOR[pu],22);
      }
    }
  }
  if(e.key==="0") {
    e.preventDefault();
    if(gameMode.value==="tron"&&!is2PAI.value&&heldSkill2) activateDuelSkill(2);
  }
}

function keyup(e:KeyboardEvent) {
  if(e.key===" ") pressedKeys.delete(" ");
}

// ── step ───────────────────────────────────────────────────
function step() {
  if(isDying) return;
  if(gameMode.value==="tron"){ stepDuel(); return; }
  if(isAI.value) aiStep();
  if(dirQueue.length) dir=dirQueue.shift()!;
  // 記錄真人移動軌跡（供 AI 學習）
  if(!isAI.value){
    const hi=(HC[snake[0]!.x]as number[])[snake[0]!.y]!;
    recordHumanMove(hi,dir,snake.length);
  }
  currentStep++;

  if(fieldPU){fieldPU.life--;if(fieldPU.life<=0)fieldPU=null;}
  activePUs=activePUs.filter(p=>{p.steps--;return p.steps>0;});

  const ghost  = activePUs.some(p=>p.type==="ghost");
  const shield = activePUs.some(p=>p.type==="shield");

  let x=snake[0]!.x, y=snake[0]!.y;
  if(dir==="UP")    y--;
  if(dir==="DOWN")  y++;
  if(dir==="LEFT")  x--;
  if(dir==="RIGHT") x++;

  if(ghost){x=(x+GRID)%GRID;y=(y+GRID)%GRID;}
  else if(x<0||y<0||x>=GRID||y>=GRID){
    if(shield){
      activePUs=activePUs.filter(p=>p.type!=="shield");
      burst(snake[0]!.x*CELL+CELL/2,snake[0]!.y*CELL+CELL/2,"#60a5fa",22);
      // 護盾反彈：方向反轉 + 往回退一格
      const oppR:Record<string,"UP"|"DOWN"|"LEFT"|"RIGHT">={UP:"DOWN",DOWN:"UP",LEFT:"RIGHT",RIGHT:"LEFT"};
      dir=oppR[dir]!;
      dirQueue.length=0;
      x=snake[0]!.x; y=snake[0]!.y;
      if(dir==="UP")y--;if(dir==="DOWN")y++;if(dir==="LEFT")x--;if(dir==="RIGHT")x++;
      snake.unshift({x,y});
      if(snake.length>1) snake.pop();
      return;
    } else { triggerDeath(); return; }
  }

  const willEat=x===food.x&&y===food.y;
  if((willEat?snake:snake.slice(0,-1)).some(s=>s.x===x&&s.y===y)){
    if(shield){
      activePUs=activePUs.filter(p=>p.type!=="shield");
      burst(x*CELL+CELL/2,y*CELL+CELL/2,"#60a5fa",22);
      return;
    } else { triggerDeath(); return; }
  }

  // trap collision (stage mode)
  if(gameMode.value==="stage"&&traps.some(t=>t.x===x&&t.y===y)){
    if(shield){
      activePUs=activePUs.filter(p=>p.type!=="shield");
      burst(x*CELL+CELL/2,y*CELL+CELL/2,"#60a5fa",22);
    } else { triggerDeath(); return; }
  }

  snake.unshift({x,y});

  if(x===food.x&&y===food.y){
    const dbl=activePUs.some(p=>p.type==="double");
    combo = currentStep-lastFoodStep<=9 ? combo+1 : 0;
    lastFoodStep=currentStep;
    const pts=Math.ceil((dbl?2:1)*(1+combo*.4));
    score.value+=pts;
    burst(x*CELL+CELL/2,y*CELL+CELL/2,"#ff4060",16);
    if(combo>0) comboText={text:`COMBO ×${combo+1}`,x:x*CELL+CELL/2,y:y*CELL,life:45};
    const nl=Math.floor(score.value/6)+1;
    if(nl>level){level=nl;levelFlash=50;}
    spawnFood(); trySpawnPU();
  } else {
    snake.pop();
  }

  if(fieldPU&&x===fieldPU.x&&y===fieldPU.y){
    const pu=fieldPU; fieldPU=null;
    burst(x*CELL+CELL/2,y*CELL+CELL/2,PU_COLOR[pu.type],22);
    if(pu.type==="shrink"){
      snake=snake.slice(0,Math.max(1,snake.length>>1));
    } else {
      const ex=activePUs.find(p=>p.type===pu.type);
      if(ex) ex.steps=PU_STEPS[pu.type];
      else activePUs.push({type:pu.type,steps:PU_STEPS[pu.type],maxSteps:PU_STEPS[pu.type]});
    }
  }

  // Boss mode logic
  if(gameMode.value==="boss"){ stepBoss(); bossFireStep(); }

  // Stage mode logic
  if(gameMode.value==="stage") stepStage();

  recordFrame();
}

// ── Boss fire helpers ──────────────────────────────────────
let missileCooldown = 0;
function fireMissile() {
  if(missiles.length>=5) return;
  if(!bossData?.active) return;
  const dx=dir==="LEFT"?-1:dir==="RIGHT"?1:0;
  const dy=dir==="UP"?-1:dir==="DOWN"?1:0;
  const h=snake[0]!;
  missiles.push({x:h.x+dx*2,y:h.y+dy*2,dx,dy,life:40});
  burst(h.x*CELL+CELL/2,h.y*CELL+CELL/2,"#fbd74b",6);
}
function aiBossFire() {
  if(!bossData?.active||missiles.length>=5) return;
  const bhead=bossData.body[0]!;
  const head=snake[0]!;
  const sameRow=head.x===bhead.x, sameCol=head.y===bhead.y;
  if((sameRow||sameCol) && Math.random()<0.4) fireMissile();
  else if(Math.random()<0.03) fireMissile();
}
function bossFireStep() {
  if(isAI.value){ aiBossFire(); return; }
  if(pressedKeys.has(" ") && missileCooldown<=0){
    fireMissile();
    missileCooldown=3;
  }
  if(missileCooldown>0) missileCooldown--;
}

// ── Stage mode step ────────────────────────────────────────
function genStageTraps(s:number) {
  const t:{x:number;y:number}[]=[];
  const count = s<=3 ? 0 : Math.min(3 + (s-3)*2, 30);
  for(let i=0;i<count;i++){
    let p:{x:number;y:number};
    let attempts=0;
    do {
      p={x:Math.floor(Math.random()*GRID),y:Math.floor(Math.random()*GRID)};
      attempts++;
    } while(attempts<50 && (
      snake.some(sn=>sn.x===p.x&&sn.y===p.y) ||
      (p.x===food.x&&p.y===food.y) ||
      t.some(tp=>tp.x===p.x&&tp.y===p.y) ||
      (portal.value&&p.x===portal.value.x&&p.y===portal.value.y) ||
      (fieldPU&&p.x===fieldPU.x&&p.y===fieldPU.y)
    ));
    t.push(p);
  }
  return t;
}
function stepStage() {
  const target=getStageTarget(stage.value);

  // generate traps on stage start (skip when portal is active)
  if(traps.length===0 && stage.value>1 && !portal.value) traps=genStageTraps(stage.value);

  // spawn portal when score >= target
  if(score.value>=target && !portal.value){
    const head=snake[0]!;
    let p:{x:number;y:number};
    let attempts=0;
    do {
      p=randPos([...snake]);
      attempts++;
    } while(attempts<50 && (Math.abs(p.x-head.x)+Math.abs(p.y-head.y))<8);
    portal.value=p;
    // clear everything — only portal remains
    food={x:-1,y:-1}; fieldPU=null; traps=[];
    burst(p.x*CELL+CELL/2,p.y*CELL+CELL/2,"#7c3aed",30);
  }

  // check portal entry
  if(portal.value){
    const head=snake[0]!;
    if(head.x===portal.value.x && head.y===portal.value.y){
      // advance stage
      const bonus = stage.value * 5;
      coins.value += bonus;
      saveShop();
      stage.value++;
      portal.value=null;
      traps=[];
      levelFlash=60;
      comboText={text:`🎉 STAGE ${stage.value-1} CLEAR! +${bonus}🪙`,x:SIZE/2,y:SIZE/3,life:90};
      if(stage.value>MAX_STAGE){
        // cleared all stages!
        comboText={text:`🏆 ALL CLEAR! 總分 ${score.value}`,x:SIZE/2,y:SIZE/3,life:120};
        setTimeout(()=>{triggerDeath();},1500);
        return;
      }
      spawnFood();
    }
  }
}

// ── Duel helpers ───────────────────────────────────────────
let fieldPU2:{x:number;y:number;type:PUType;life:number}|null = null;
let activePUs2:ActivePU[] = [];

function getCurrentGrid(player:1|2):1|2 {
  if(player===1) return inOpponentGrid1 ? 2 : 1;
  return inOpponentGrid2 ? 1 : 2;
}

function activateDuelSkill(p:1|2) {
  const skill = p===1 ? heldSkill1 : heldSkill2;
  if(!skill) return;
  if(skill==="shrink"){
    if(p===1) snake=snake.slice(0,Math.max(1,snake.length>>1));
    else snake2=snake2.slice(0,Math.max(1,snake2.length>>1));
  } else if(skill==="trap"){
    if(p===1){
      const tp=randPos([...snake2,...traps2,...(portal2?[portal2]:[]),food2]);
      traps2.push({...tp});
    } else if(p===2){
      const tp=randPos([...snake,...traps1,...(portal1?[portal1]:[]),food]);
      traps1.push({...tp});
    }
  } else if(skill==="remove-trap"){
    if(p===1) traps1=[];
    else traps2=[];
  } else {
    const arr=p===1?activePUs:activePUs2;
    const ex=arr.find(a=>a.type===skill);
    if(ex) ex.steps=PU_STEPS[skill];
    else arr.push({type:skill,steps:PU_STEPS[skill],maxSteps:PU_STEPS[skill]});
  }
  if(p===1) heldSkill1=null; else heldSkill2=null;
}

function spawnDuelFood(g:1|2) {
  if(g===1) food=randPos([...snake,...traps1,...(fieldPU?[fieldPU]:[]),...(portal1?[portal1]:[])]);
  else food2=randPos([...snake2,...traps2,...(fieldPU2?[fieldPU2]:[]),...(portal2?[portal2]:[])]);
}
function trySpawnDuelPU(g:1|2) {
  const s = g===1 ? fieldPU : fieldPU2;
  if(s) return;
  if(Math.random()<0.2){
    const types:PUType[]=[...PU_TYPES,"trap","remove-trap"];
    const type=types[Math.floor(Math.random()*types.length)]!;
    const pos=g===1
      ? randPos([...snake,food,...traps1,...(portal1?[portal1]:[])])
      : randPos([...snake2,food2,...traps2,...(portal2?[portal2]:[])]);
    if(g===1) fieldPU={...pos,type,life:22};
    else fieldPU2={...pos,type,life:22};
  }
}
function spawnDuelPortal(g:1|2) {
  if(g===1){
    if(portal1) return;
    const p=randPos([...snake,...traps1,...(fieldPU?[fieldPU]:[]),...(portal1?[portal1]:[]),food]);
    portal1=p;
    portalCD1=60+Math.floor(Math.random()*50);
  } else {
    if(portal2) return;
    const p=randPos([...snake2,...traps2,...(fieldPU2?[fieldPU2]:[]),...(portal2?[portal2]:[]),food2]);
    portal2=p;
    portalCD2=60+Math.floor(Math.random()*50);
  }
}
function duelRespawn(p:1|2) {
  if(p===1){
    snake=[{x:3,y:10},{x:2,y:10},{x:1,y:10}]; dir="RIGHT";
    inOpponentGrid1=false; opponentTimer1=0; heldSkill1=null;
  } else {
    snake2=[{x:16,y:10},{x:17,y:10},{x:18,y:10}]; dir2="LEFT";
    inOpponentGrid2=false; opponentTimer2=0; heldSkill2=null;
  }
}

// ── Duel step (雙人對戰) ─────────────────────────────────
function stepDuel() {
  if(is2PAI.value) duelAI();
  // consume direction queue
  if(dirQueue.length) dir=dirQueue.shift()!;
  if(dirQueue2.length) dir2=dirQueue2.shift()!;
  const dv:Record<string,{x:number;y:number}>={UP:{x:0,y:-1},DOWN:{x:0,y:1},LEFT:{x:-1,y:0},RIGHT:{x:1,y:0}};

  // ── Decay timers ──
  if(fieldPU){fieldPU.life--;if(fieldPU.life<=0)fieldPU=null;}
  if(fieldPU2){fieldPU2.life--;if(fieldPU2.life<=0)fieldPU2=null;}
  activePUs=activePUs.filter(p=>{p.steps--;return p.steps>0;});
  activePUs2=activePUs2.filter(p=>{p.steps--;return p.steps>0;});

  const h1=snake[0]!, h2=snake2[0]!;
  const nx1=h1.x+dv[dir]!.x, ny1=h1.y+dv[dir]!.y;
  const nx2=h2.x+dv[dir2]!.x, ny2=h2.y+dv[dir2]!.y;

  const g1=getCurrentGrid(1), g2=getCurrentGrid(2);
  const ghost1=activePUs.some(p=>p.type==="ghost");
  const ghost2=activePUs2.some(p=>p.type==="ghost");
  const shield1=activePUs.some(p=>p.type==="shield");
  const shield2=activePUs2.some(p=>p.type==="shield");

  // ── Collision ──
  let dead1=false, dead2=false;
  let shieldBlocked1=false, shieldBlocked2=false;

  function checkWall(nx:number,ny:number,ghost:boolean):boolean{
    if(ghost) return false;
    return nx<0||ny<0||nx>=GRID||ny>=GRID;
  }
  function consumeShield(p:1|2):boolean{
    if(p===1&&shield1){activePUs=activePUs.filter(a=>a.type!=="shield");return true;}
    if(p===2&&shield2){activePUs2=activePUs2.filter(a=>a.type!=="shield");return true;}
    return false;
  }

  // wall
  if(!dead1&&checkWall(nx1,ny1,ghost1)){
    if(!consumeShield(1)) dead1=true; else shieldBlocked1=true;
  }
  if(!dead2&&checkWall(nx2,ny2,ghost2)){
    if(!consumeShield(2)) dead2=true; else shieldBlocked2=true;
  }

  // food (先算，自撞才知道尾巴會不會移開)
  const myFood1=g1===1?food:food2;
  const myFood2=g2===1?food:food2;
  const eat1=nx1===myFood1.x&&ny1===myFood1.y;
  const eat2=nx2===myFood2.x&&ny2===myFood2.y;

  // self collision（沒吃食物→尾巴會移開，不算障礙）
  if(!dead1&&(eat1?snake:snake.slice(0,-1)).some(s=>s.x===nx1&&s.y===ny1)){
    if(!consumeShield(1)) dead1=true;
  }
  if(!dead2&&(eat2?snake2:snake2.slice(0,-1)).some(s=>s.x===nx2&&s.y===ny2)){
    if(!consumeShield(2)) dead2=true;
  }

  // cross collision (only if in same grid)
  if(g1===g2){
    if(!dead1&&snake2.some(s=>s.x===nx1&&s.y===ny1)&&!consumeShield(1)) dead1=true;
    if(!dead2&&snake.some(s=>s.x===nx2&&s.y===ny2)&&!consumeShield(2)) dead2=true;
    if(nx1===nx2&&ny1===ny2){dead1=true;dead2=true;}
  }

  // trap collision
  const trapsIn1=g1===1?traps1:traps2;
  const trapsIn2=g2===1?traps2:traps1;
  if(!dead1&&trapsIn1.some(t=>t.x===nx1&&t.y===ny1)){
    if(!consumeShield(1)) dead1=true;
  }
  if(!dead2&&trapsIn2.some(t=>t.x===nx2&&t.y===ny2)){
    if(!consumeShield(2)) dead2=true;
  }

  // ── Handle deaths ──
  if(dead1||dead2){
    if(dead1){
      // check if killed by opponent
      const killedByP2=g1===g2&&snake2.some(s=>s.x===nx1&&s.y===ny1);
      lives1--;
      if(killedByP2){lives2=Math.min(3,lives2+1);score2++;}
      if(!killedByP2) score1=Math.max(0,score1-1);
    }
    if(dead2){
      // AI 死亡學習：記錄致命位置+方向
      if(is2PAI.value&&h2){
        const di=(HC[h2.x]as number[])[h2.y]!;
        const obs2=new Set<string>();
        for(const s of snake2)obs2.add(`${s.x},${s.y}`);
        if(g1===g2)for(const s of snake)obs2.add(`${s.x},${s.y}`);
        const q2:[number,number][]=[[nx2,ny2]];const vis2=new Set(obs2);vis2.add(`${nx2},${ny2}`);let sp2=0;
        while(q2.length){const[ax,ay]=q2.shift()!;sp2++;for(const dd of["UP","DOWN","LEFT","RIGHT"] as const){
          let bx=ax+dv[dd]!.x,by=ay+dv[dd]!.y;
          if(ghost2){bx=(bx+GRID)%GRID;by=(by+GRID)%GRID;}else if(bx<0||by<0||bx>=GRID||by>=GRID)continue;
          const bk=`${bx},${by}`;if(!vis2.has(bk)){vis2.add(bk);q2.push([bx,by]);}
        }}
        recordAIDeath(di,dir2,snake2.length,sp2);
      }
      const killedByP1=g1===g2&&snake.some(s=>s.x===nx2&&s.y===ny2);
      lives2--;
      if(killedByP1){lives1=Math.min(3,lives1+1);score1++;}
      if(!killedByP1) score2=Math.max(0,score2-1);
    }

    if(lives1<=0||lives2<=0){
      isDying=true;
      if(lives1<=0&&lives2<=0) tronWinner.value="🤝 DRAW";
      else if(lives1<=0) {
        if(is2PAI.value){ tronWinner.value=`🤖：${AI_TRASH[Math.floor(Math.random()*AI_TRASH.length)]}`; }
        else { tronWinner.value="🏆 P2 勝利"; }
      }
      else tronWinner.value="🏆 P1 勝利";
      shakeFrames=24;
      if(dead1) snake.forEach(s=>burst((g1===1?0:DUEL_GW)+s.x*DUEL_CW+DUEL_CW/2,s.y*DUEL_CH+DUEL_CH/2,"#ff4060",4));
      if(dead2) snake2.forEach(s=>burst((g2===1?0:DUEL_GW)+s.x*DUEL_CW+DUEL_CW/2,s.y*DUEL_CH+DUEL_CH/2,"#c084fc",4));
      setTimeout(()=>{paused.value=false;state.value="gameover";},700);
      return;
    }

    // respawn
    burst((g1===1?0:DUEL_GW)+h1.x*DUEL_CW+DUEL_CW/2,h1.y*DUEL_CH+DUEL_CH/2,dead1?"#ff4060":"#c084fc",14);
    if(dead1) duelRespawn(1);
    if(dead2) duelRespawn(2);
    return;
  }

  // ── Move snakes ──
  if(shieldBlocked1){
    const oppR:Record<string,"UP"|"DOWN"|"LEFT"|"RIGHT">={UP:"DOWN",DOWN:"UP",LEFT:"RIGHT",RIGHT:"LEFT"};
    dir=oppR[dir]!; dirQueue.length=0;
    const bx=h1.x-(nx1<0?1:nx1>=GRID?-1:0);
    const by=h1.y-(ny1<0?1:ny1>=GRID?-1:0);
    snake.unshift({x:bx>=0&&bx<GRID?bx:h1.x,y:by>=0&&by<GRID?by:h1.y});
  } else {
    snake.unshift({x:nx1,y:ny1});
  }
  if(shieldBlocked2){
    const oppR2:Record<string,"UP"|"DOWN"|"LEFT"|"RIGHT">={UP:"DOWN",DOWN:"UP",LEFT:"RIGHT",RIGHT:"LEFT"};
    dir2=oppR2[dir2]!; dirQueue2.length=0;
    const bx=h2.x-(nx2<0?1:nx2>=GRID?-1:0);
    const by=h2.y-(ny2<0?1:ny2>=GRID?-1:0);
    snake2.unshift({x:bx>=0&&bx<GRID?bx:h2.x,y:by>=0&&by<GRID?by:h2.y});
  } else {
    snake2.unshift({x:nx2,y:ny2});
  }

  // ── Food ──
  if(eat1){score1++;burst((g1===1?0:DUEL_GW)+nx1*DUEL_CW+DUEL_CW/2,ny1*DUEL_CH+DUEL_CH/2,"#ff4060",10);spawnDuelFood(g1);}
  if(eat2){score2++;burst((g2===1?0:DUEL_GW)+nx2*DUEL_CW+DUEL_CW/2,ny2*DUEL_CH+DUEL_CH/2,"#c084fc",10);spawnDuelFood(g2);}
  if(!eat1&&!shieldBlocked1) snake.pop();
  if(!eat2&&!shieldBlocked2) snake2.pop();

  // ── Collect PU ──
  const puHere1=g1===1?fieldPU:fieldPU2;
  const puHere2=g2===1?fieldPU:fieldPU2;
  if(puHere1&&nx1===puHere1.x&&ny1===puHere1.y){
    heldSkill1=puHere1.type;
    const store=g1===1?fieldPU:fieldPU2;
    if(store)store.life=0;
    burst((g1===1?0:DUEL_GW)+nx1*DUEL_CW+DUEL_CW/2,ny1*DUEL_CH+DUEL_CH/2,PU_COLOR[puHere1.type],14);
  }
  if(puHere2&&nx2===puHere2.x&&ny2===puHere2.y){
    heldSkill2=puHere2.type;
    const store=g2===1?fieldPU:fieldPU2;
    if(store)store.life=0;
    burst((g2===1?0:DUEL_GW)+nx2*DUEL_CW+DUEL_CW/2,ny2*DUEL_CH+DUEL_CH/2,PU_COLOR[puHere2.type],14);
  }

  // ── Portal ──
  const portalIn1=g1===1?portal1:portal2;
  const portalIn2=g2===1?portal1:portal2;
  if(portalIn1&&nx1===portalIn1.x&&ny1===portalIn1.y){
    inOpponentGrid1=!inOpponentGrid1;
    opponentTimer1=OPPONENT_STEPS;
    burst((g1===1?0:DUEL_GW)+nx1*DUEL_CW+DUEL_CW/2,ny1*DUEL_CH+DUEL_CH/2,"#7c3aed",20);
  }
  if(portalIn2&&nx2===portalIn2.x&&ny2===portalIn2.y){
    inOpponentGrid2=!inOpponentGrid2;
    opponentTimer2=OPPONENT_STEPS;
    burst((g2===1?0:DUEL_GW)+nx2*DUEL_CW+DUEL_CW/2,ny2*DUEL_CH+DUEL_CH/2,"#7c3aed",20);
  }

  // ── Opponent return timer ──
  if(inOpponentGrid1){
    opponentTimer1--;
    if(opponentTimer1<=0){
      inOpponentGrid1=false;
      const sp=randPos([...snake2,...traps1,food]);
      snake[0]=sp;
      burst(sp.x*DUEL_CW+DUEL_CW/2,sp.y*DUEL_CH+DUEL_CH/2,"#a78bfa",18);
    }
  }
  if(inOpponentGrid2){
    opponentTimer2--;
    if(opponentTimer2<=0){
      inOpponentGrid2=false;
      const sp=randPos([...snake,...traps2,food2]);
      snake2[0]=sp;
      burst(DUEL_GW+sp.x*DUEL_CW+DUEL_CW/2,sp.y*DUEL_CH+DUEL_CH/2,"#a78bfa",18);
    }
  }

  // ── Spawn stuff ──
  portalCD1--; portalCD2--;
  // traps are permanent (use CLEANSE skill to remove)
  // grid1
  if(!portal1&&portalCD1<=0) spawnDuelPortal(1);
  if(portal1&&Math.random()<0.005){portal1=null;}
  trySpawnDuelPU(1);
  // grid2
  if(!portal2&&portalCD2<=0) spawnDuelPortal(2);
  if(portal2&&Math.random()<0.005){portal2=null;}
  trySpawnDuelPU(2);
}

// ── Duel AI（攻擊型 AI：技能 → 撿道具 → 攻擊 → 吃食物 → 追擊 → 生存）──
function duelAI() {
  const head = snake2[0]!;
  const tail = snake2[snake2.length - 1]!;
  const directionVectors = {
    UP: { x: 0, y: -1 }, DOWN: { x: 0, y: 1 },
    LEFT: { x: -1, y: 0 }, RIGHT: { x: 1, y: 0 }
  } as const;
  const oppositeDirection = {
    UP: "DOWN", DOWN: "UP", LEFT: "RIGHT", RIGHT: "LEFT"
  } as const;
  const allDirections = ["UP", "DOWN", "LEFT", "RIGHT"] as const;

  const ghostActive = !!activePUs2.find(p => p.type === "ghost") &&
    activePUs2.find(p => p.type === "ghost")!.steps > 1;
  const inOpponentGrid = inOpponentGrid2;
  const sameGrid = getCurrentGrid(1) === getCurrentGrid(2);
  const myTraps = inOpponentGrid ? traps1 : traps2;
  const myFood = inOpponentGrid ? food : food2;
  const foodExists = !!myFood && !(inOpponentGrid && myFood.x === -1);
  const playerHead = snake[0]!;
  const playerDir = dir;

  // ── Helper: build obstacle set (without tail since non-eating moves pop it) ──
  function buildObstaclesWithoutTail(): Set<string> {
    const obstacles = new Set<string>();
    for (const segment of snake2) obstacles.add(`${segment.x},${segment.y}`);
    if (sameGrid) for (const segment of snake) obstacles.add(`${segment.x},${segment.y}`);
    for (const trap of myTraps) obstacles.add(`${trap.x},${trap.y}`);
    obstacles.delete(`${tail.x},${tail.y}`);
    return obstacles;
  }

  // ── Helper: build obstacle set with tail (for skill activation checks) ──
  function buildObstaclesWithTail(): Set<string> {
    const obstacles = new Set<string>();
    for (const segment of snake2) obstacles.add(`${segment.x},${segment.y}`);
    if (sameGrid) for (const segment of snake) obstacles.add(`${segment.x},${segment.y}`);
    for (const trap of myTraps) obstacles.add(`${trap.x},${trap.y}`);
    return obstacles;
  }

  // ── Breadth-First Search: returns array of direction strings, or null if unreachable ──
  function breadthFirstSearch(
    startX: number, startY: number,
    targetX: number, targetY: number,
    obstacles: Set<string>
  ): string[] | null {
    const visited = new Set<string>();
    visited.add(`${startX},${startY}`);
    type QueueNode = { x: number; y: number; path: string[] };
    const queue: QueueNode[] = [{ x: startX, y: startY, path: [] }];
    while (queue.length > 0) {
      const current = queue.shift()!;
      if (current.x === targetX && current.y === targetY) return current.path;
      for (const direction of allDirections) {
        if (current.path.length === 0 && direction === oppositeDirection[dir2]) continue;
        let nextX = current.x + directionVectors[direction].x;
        let nextY = current.y + directionVectors[direction].y;
        if (ghostActive) {
          nextX = (nextX + GRID) % GRID;
          nextY = (nextY + GRID) % GRID;
        } else if (nextX < 0 || nextY < 0 || nextX >= GRID || nextY >= GRID) {
          continue;
        }
        const neighborKey = `${nextX},${nextY}`;
        if (visited.has(neighborKey)) continue;
        if (obstacles.has(neighborKey)) continue;
        if (!ghostActive && myTraps.some(trap => trap.x === nextX && trap.y === nextY)) continue;
        visited.add(neighborKey);
        queue.push({ x: nextX, y: nextY, path: [...current.path, direction] });
      }
    }
    return null;
  }

  // ════════════════════════════════════════════════════════════
  //  STEP 0: Skill Usage — activate held skills strategically
  //  Priority: double > shrink > trap > remove-trap > shield > ghost > slow
  // ════════════════════════════════════════════════════════════
  if (heldSkill2) {
    const skill = heldSkill2;
    let shouldUse = false;
    if (skill === "double") {
      shouldUse = true;
    } else if (skill === "shrink") {
      if (snake2.length > 8) shouldUse = true;
    } else if (skill === "trap" && sameGrid) {
      shouldUse = true;
    } else if (skill === "remove-trap") {
      if (myTraps.length > 0) shouldUse = true;
    } else if (skill === "shield") {
      const obs = buildObstaclesWithTail();
      let safeNeighbors = 0;
      for (const d of allDirections) {
        let nx = head.x + directionVectors[d].x;
        let ny = head.y + directionVectors[d].y;
        if (ghostActive) { nx = (nx + GRID) % GRID; ny = (ny + GRID) % GRID; }
        else if (nx < 0 || ny < 0 || nx >= GRID || ny >= GRID) continue;
        if (!obs.has(`${nx},${ny}`)) safeNeighbors++;
      }
      if (safeNeighbors < 2) shouldUse = true;
    } else if (skill === "ghost") {
      const obs = buildObstaclesWithTail();
      let safeNeighbors = 0;
      for (const d of allDirections) {
        let nx = head.x + directionVectors[d].x;
        let ny = head.y + directionVectors[d].y;
        if (nx < 0 || ny < 0 || nx >= GRID || ny >= GRID) continue;
        if (!obs.has(`${nx},${ny}`)) safeNeighbors++;
      }
      if (safeNeighbors < 1) shouldUse = true;
      if (sameGrid && !shouldUse) shouldUse = true;
    } else if (skill === "slow") {
      if (sameGrid) shouldUse = true;
    }
    if (shouldUse) {
      if (skill === "trap") {
        // Smart trap placement: put trap in front of player's direction
        const trapX = playerHead.x + directionVectors[playerDir].x;
        const trapY = playerHead.y + directionVectors[playerDir].y;
        if (trapX >= 0 && trapY >= 0 && trapX < GRID && trapY < GRID &&
            !snake.some(s => s.x === trapX && s.y === trapY) &&
            !traps1.some(t => t.x === trapX && t.y === trapY)) {
          traps1.push({x: trapX, y: trapY});
          heldSkill2 = null;
        } else {
          activateDuelSkill(2);
        }
      } else {
        activateDuelSkill(2);
      }
    }
  }

  // ════════════════════════════════════════════════════════════
  //  STEP 0.5: Collect useful field PU items (shield / ghost / trap)
  //  BFS → Virtual Snake → Tail Connectivity (same logic as food)
  // ════════════════════════════════════════════════════════════
  const myFieldPU = inOpponentGrid ? fieldPU : fieldPU2;
  if (myFieldPU && (myFieldPU.type === "shield" || myFieldPU.type === "ghost" || myFieldPU.type === "trap")) {
    const pathToPU = breadthFirstSearch(
      head.x, head.y,
      myFieldPU.x, myFieldPU.y,
      buildObstaclesWithoutTail()
    );
    if (pathToPU !== null && pathToPU.length > 0) {
      const nonEatingSteps = Math.min(pathToPU.length, snake2.length - 1);
      const virtualTailIndex = snake2.length - 1 - nonEatingSteps;
      const virtualTail = snake2[virtualTailIndex]!;
      const obstaclesAfterMove = new Set<string>();
      for (let i = 0; i < virtualTailIndex; i++) {
        obstaclesAfterMove.add(`${snake2[i]!.x},${snake2[i]!.y}`);
      }
      obstaclesAfterMove.delete(`${virtualTail.x},${virtualTail.y}`);
      const canReachTail = breadthFirstSearch(
        myFieldPU.x, myFieldPU.y,
        virtualTail.x, virtualTail.y,
        obstaclesAfterMove
      );
      if (canReachTail !== null) {
        dir2 = pathToPU[0] as "UP" | "DOWN" | "LEFT" | "RIGHT";
        return;
      }
    }
  }

  // ════════════════════════════════════════════════════════════
  //  STEP 0.75: Attack / Intercept Player
  //  When on same grid and player is within 10 BFS steps,
  //  chase their head with safety check.
  // ════════════════════════════════════════════════════════════
  if (sameGrid) {
    const pathToPlayer = breadthFirstSearch(
      head.x, head.y,
      playerHead.x, playerHead.y,
      buildObstaclesWithoutTail()
    );
    if (pathToPlayer !== null && pathToPlayer.length > 0 && pathToPlayer.length <= 10) {
      const nonEatingSteps = Math.min(pathToPlayer.length, snake2.length - 1);
      const virtualTailIndex = snake2.length - 1 - nonEatingSteps;
      const virtualTail = snake2[virtualTailIndex]!;
      const obstaclesAfterMove = new Set<string>();
      for (let i = 0; i < virtualTailIndex; i++) {
        obstaclesAfterMove.add(`${snake2[i]!.x},${snake2[i]!.y}`);
      }
      obstaclesAfterMove.delete(`${virtualTail.x},${virtualTail.y}`);
      const canReachTail = breadthFirstSearch(
        playerHead.x, playerHead.y,
        virtualTail.x, virtualTail.y,
        obstaclesAfterMove
      );
      if (canReachTail !== null) {
        dir2 = pathToPlayer[0] as "UP" | "DOWN" | "LEFT" | "RIGHT";
        return;
      }
    }
  }

  // ════════════════════════════════════════════════════════════
  //  STEP 1 + 2 + 3: Shortest Path to Food → Virtual Snake
  //  → Tail Connectivity
  // ════════════════════════════════════════════════════════════
  if (foodExists) {
    const pathToFood = breadthFirstSearch(
      head.x, head.y,
      myFood!.x, myFood!.y,
      buildObstaclesWithoutTail()
    );
    if (pathToFood !== null && pathToFood.length > 0) {
      const nonEatingSteps = Math.min(pathToFood.length - 1, snake2.length - 1);
      const virtualTailIndex = snake2.length - 1 - nonEatingSteps;
      const virtualTail = snake2[virtualTailIndex]!;
      const obstaclesAfterEating = new Set<string>();
      for (let i = 0; i < virtualTailIndex; i++) {
        const segment = snake2[i]!;
        obstaclesAfterEating.add(`${segment.x},${segment.y}`);
      }
      obstaclesAfterEating.delete(`${virtualTail.x},${virtualTail.y}`);
      const canReachTail = breadthFirstSearch(
        myFood!.x, myFood!.y,
        virtualTail.x, virtualTail.y,
        obstaclesAfterEating
      );
      if (canReachTail !== null) {
        dir2 = pathToFood[0] as "UP" | "DOWN" | "LEFT" | "RIGHT";
        return;
      }
    }
  }

  // ════════════════════════════════════════════════════════════
  //  STEP 3.5: Chase Player (fallback, same grid only)
  //  No safety check — this is better than random wandering
  // ════════════════════════════════════════════════════════════
  if (sameGrid) {
    const pathToPlayer = breadthFirstSearch(
      head.x, head.y,
      playerHead.x, playerHead.y,
      buildObstaclesWithoutTail()
    );
    if (pathToPlayer !== null && pathToPlayer.length > 0) {
      dir2 = pathToPlayer[0] as "UP" | "DOWN" | "LEFT" | "RIGHT";
      return;
    }
  }

  // ════════════════════════════════════════════════════════════
  //  STEP 4 (Fallback): Longest Path to Tail — evaluate each
  //  candidate direction and pick the one with the longest
  //  safe path to the tail.
  // ════════════════════════════════════════════════════════════
  const obstacles = buildObstaclesWithoutTail();
  let bestDirection: "UP" | "DOWN" | "LEFT" | "RIGHT" | null = null;
  let longestPathLength = -1;
  for (const direction of allDirections) {
    if (direction === oppositeDirection[dir2]) continue;
    let nextHeadX = head.x + directionVectors[direction].x;
    let nextHeadY = head.y + directionVectors[direction].y;
    if (ghostActive) {
      nextHeadX = (nextHeadX + GRID) % GRID;
      nextHeadY = (nextHeadY + GRID) % GRID;
    } else if (nextHeadX < 0 || nextHeadY < 0 || nextHeadX >= GRID || nextHeadY >= GRID) {
      continue;
    }
    if (obstacles.has(`${nextHeadX},${nextHeadY}`)) continue;
    if (!ghostActive && myTraps.some(trap => trap.x === nextHeadX && trap.y === nextHeadY)) continue;
    const pathToTail = breadthFirstSearch(nextHeadX, nextHeadY, tail.x, tail.y, obstacles);
    if (pathToTail !== null && pathToTail.length > longestPathLength) {
      longestPathLength = pathToTail.length;
      bestDirection = direction;
    }
  }
  if (bestDirection !== null) {
    dir2 = bestDirection;
    return;
  }

  // ── Emergency fallback: any non-reverse safe direction ──
  for (const direction of allDirections) {
    if (direction === oppositeDirection[dir2]) continue;
    let nextHeadX = head.x + directionVectors[direction].x;
    let nextHeadY = head.y + directionVectors[direction].y;
    if (ghostActive) {
      nextHeadX = (nextHeadX + GRID) % GRID;
      nextHeadY = (nextHeadY + GRID) % GRID;
    } else if (nextHeadX < 0 || nextHeadY < 0 || nextHeadX >= GRID || nextHeadY >= GRID) {
      continue;
    }
    if (!obstacles.has(`${nextHeadX},${nextHeadY}`)) {
      dir2 = direction;
      return;
    }
  }
}



// ── Boss step ─────────────────────────────────────────────
function stepBoss() {
  // missiles
  missiles=missiles.filter(m=>{m.x+=m.dx;m.y+=m.dy;m.life--;const w=m.x<0||m.y<0||m.x>=GRID||m.y>=GRID;return m.life>0&&!w;});

  // spawn boss
  if(!bossData&&score.value>=BOSS_THRESHOLD){
    const spots:{x:number;y:number}[]=[];
    for(let i=0;i<GRID;i++)for(let j=0;j<GRID;j++){
      if(!snake.some(s=>s.x===i&&s.y===j)&&!(i===food.x&&j===food.y))
        spots.push({x:i,y:j});
    }
    if(spots.length){
      const p=spots[Math.floor(Math.random()*spots.length)]!;
      bossData={
        body:Array.from({length:12},(_,k)=>({x:p.x-k,y:p.y})),
        dir:"LEFT",health:20,maxHealth:20,active:true,alert:60,
      };
      levelFlash=60;
    }
  }

  if(!bossData?.active) return;
  if(bossData.alert>0){bossData.alert--;return;}

  // boss AI: chase player
  const bh=bossData.body[0]!;
  const dv:Record<string,{x:number;y:number}>={UP:{x:0,y:-1},DOWN:{x:0,y:1},LEFT:{x:-1,y:0},RIGHT:{x:1,y:0}};
  const OPP:Record<string,string>={UP:"DOWN",DOWN:"UP",LEFT:"RIGHT",RIGHT:"LEFT"};
  const ALL:("UP"|"DOWN"|"LEFT"|"RIGHT")[]=["UP","DOWN","LEFT","RIGHT"];

  const allOcc=new Set<string>();
  for(const s of snake) allOcc.add(`${s.x},${s.y}`);
  for(const s of bossData.body) allOcc.add(`${s.x},${s.y}`);
  allOcc.delete(`${bh.x},${bh.y}`);

  // prefer direction toward player
  const target=snake[0]!;
  const px=bh.x-target.x, py=bh.y-target.y;
  const preferred=Math.abs(px)>Math.abs(py)
    ?(px>0?"LEFT":"RIGHT")
    :(py>0?"UP":"DOWN");

  let moved=false;
  for(const d of [preferred,...ALL] as ("UP"|"DOWN"|"LEFT"|"RIGHT")[]){
    if(d===OPP[bossData.dir]) continue;
    const nx=bh.x+dv[d]!.x, ny=bh.y+dv[d]!.y;
    if(nx<0||ny<0||nx>=GRID||ny>=GRID){bossData.health-=2;burst(nx<0?0:nx>=GRID?SIZE:bh.x*CELL+CELL/2,ny<0?0:ny>=GRID?SIZE:bh.y*CELL+CELL/2,"#ff4060",10);continue;}
    if(allOcc.has(`${nx},${ny}`)) continue;
    bossData.dir=d; moved=true; break;
  }
  if(!moved){
    // boss stuck — damage
    bossData.health-=3;
    burst(bh.x*CELL+CELL/2,bh.y*CELL+CELL/2,"#ff4060",12);
  }

  // move boss
  const d=dv[bossData.dir]!;
  const nh=bh.x+d.x, nyh=bh.y+d.y;
  bossData.body.unshift({x:nh,y:nyh});
  bossData.body.pop();

  // missile hit boss
  for(let mi=missiles.length-1;mi>=0;mi--){
    const m=missiles[mi]!;
    if(bossData.body.some(s=>s.x===m.x&&s.y===m.y)){
      bossData.health-=3;
      burst(m.x*CELL+CELL/2,m.y*CELL+CELL/2,"#fbd74b",16);
      missiles.splice(mi,1);
    }
  }

  // player hits boss body → death
  const ph=snake[0]!;
  if(bossData.body.some(s=>s.x===ph.x&&s.y===ph.y)){
    triggerDeath(); return;
  }

  // boss defeated
  if(bossData.health<=0){
    const bonus=30+bossKills.value*10;
    score.value+=bonus;
    bossKills.value++;
    for(const s of bossData.body) burst(s.x*CELL+CELL/2,s.y*CELL+CELL/2,"#fbd74b",3);
    comboText={text:`💥 BOSS 擊敗！+${bonus}`,x:SIZE/2,y:SIZE/3,life:90};
    bossData=null;
  }
}

const heldPU = ref<PUType|null>(null);
const showRevivePrompt = ref(false);

function triggerDeath() {
  isDying=true; shakeFrames=24;
  snake.forEach(s=>burst(s.x*CELL+CELL/2,s.y*CELL+CELL/2,"#ff4060",5));
  // check extra life before game over
  if(!extraLifeUsed && ownedItems.value.has("extraLife") && gameMode.value!=="tron"){
    showRevivePrompt.value=true;
    return;
  }
  setTimeout(gameOver,700);
}
function reviveYes() {
  showRevivePrompt.value=false;
  extraLifeUsed=true;
  ownedItems.value.delete("extraLife"); saveShop();
  isDying=false; shakeFrames=0;
  const h=snake[0]!;
  snake=[{x:h.x,y:h.y},{x:Math.max(0,h.x-1),y:h.y},{x:Math.max(0,h.x-2),y:h.y}];
  comboText={text:`❤️ 復活！`,x:SIZE/2,y:SIZE/3,life:60};
}
function reviveNo() {
  showRevivePrompt.value=false;
  setTimeout(gameOver,700);
}

function gameOver() {
  const mode=gameMode.value;
  let name=isAI.value?"🤖 AI":playerName.value.trim();
  let finalScore=score.value;

  if(mode==="tron"){
    name=tronWinner.value||"";
    finalScore=combo;
  }

  if(mode==="stage") finalScore=stage.value;

  if(mode!=="tron") addLB(mode,name||"匿名",finalScore).then(()=>{});
  loadHist(mode);
  history.value.unshift({label:name||"匿名",score:finalScore,frames:JSON.parse(JSON.stringify(frames)),mode});
  history.value=history.value.slice(0,10);
  if(mode!=="tron") saveHist(mode);
  // 儲存真人高手軌跡（供 AI 學習）
  if(!isAI.value&&mode!=="tron"&&finalScore>=30) saveHumanTrajectory();

  paused.value=false;
  state.value="gameover";
}

// ── replay ─────────────────────────────────────────────────
async function playReplay(item:HistoryItem) {
  if(replayTimer) clearTimeout(replayTimer);
  state.value="playing";
  if(!(await initCanvas())) return;
  particles=[]; isDying=false; paused.value=false;
  const savedAI=isAI.value; isAI.value=false;
  const savedMode=gameMode.value;
  if(item.mode) gameMode.value=item.mode as any;
  let i=0;
  const run=()=>{
    if(state.value!=="playing"){isAI.value=savedAI;gameMode.value=savedMode;return;}
    // pause support for replay
    if(paused.value){replayTimer=setTimeout(run,100);return;}
    const f=item.frames[i];
    if(!f){state.value="menu";isAI.value=savedAI;gameMode.value=savedMode;return;}
    snake=JSON.parse(JSON.stringify(f.snake));
    food={...f.food};
    fieldPU = f.fieldPU ? {...f.fieldPU} : null;
    activePUs=JSON.parse(JSON.stringify(f.activePUs));
    score.value=f.score;
    bossData=f.bossData?JSON.parse(JSON.stringify(f.bossData)):null;
    missiles=JSON.parse(JSON.stringify(f.missiles||[]));
    if(f.portal!==undefined) portal.value=f.portal?{...f.portal}:null;
    if(f.traps!==undefined) traps=JSON.parse(JSON.stringify(f.traps));
    if(f.stage!==undefined) stage.value=f.stage;
    draw();
    i++;
    replayTimer=setTimeout(run,80);
  };
  run();
}

// ── loop ───────────────────────────────────────────────────
function loop(t:number) {
  if(state.value!=="playing"){running=false;return;}
  if(!ctx){requestAnimationFrame(loop);return;}
  if(!paused.value){
    const d=t-last; last=t; acc+=d; tick++;
    particles=particles.filter(p=>p.life>0);
    particles.forEach(p=>{p.x+=p.vx;p.y+=p.vy;p.vy+=0.12;p.vx*=0.95;p.life--;});
    if(comboText) comboText.life--;
    if(levelFlash>0) levelFlash--;
    if(acc>interval()&&!isDying){acc=0;step();}
  } else {
    last=t; // reset last so no jump when unpausing
  }
  draw();
  requestAnimationFrame(loop);
}

// ── draw ───────────────────────────────────────────────────
function draw() {
  if(!ctx) return;
  const sx=shakeFrames>0?(Math.random()-.5)*7:0;
  const sy=shakeFrames>0?(Math.random()-.5)*5:0;
  if(shakeFrames>0) shakeFrames--;

  ctx.save();
  ctx.translate(sx,sy);

  const cw=gameMode.value==="tron"?SIZE*2:SIZE;
  const ch=gameMode.value==="tron"?SIZE+80:SIZE;
  ctx.fillStyle="#050a18";
  ctx.fillRect(-15,-15,cw+30,ch+30);

  if(gameMode.value==="tron"){
    drawDuel();
  } else {
    // single-player grid（加粗邊界格線）
    ctx.strokeStyle="rgba(0,255,170,.12)";
    ctx.lineWidth=1;
    for(let i=0;i<=GRID;i++){
      ctx.beginPath();ctx.moveTo(i*CELL,0);ctx.lineTo(i*CELL,SIZE);ctx.stroke();
      ctx.beginPath();ctx.moveTo(0,i*CELL);ctx.lineTo(SIZE,i*CELL);ctx.stroke();
    }
    // strong border glow（邊界加亮）
    ctx.shadowBlur=35; ctx.shadowColor="#00ffaa";
    ctx.strokeStyle="rgba(0,255,170,.6)"; ctx.lineWidth=4;
    ctx.strokeRect(0,0,SIZE,SIZE);
    ctx.shadowBlur=0;
    drawFood();
    if(fieldPU) drawPowerUpItem(fieldPU);
    drawSnake();
    if(gameMode.value==="boss"){ drawBoss(); drawMissiles(); }
    if(gameMode.value==="stage") drawStage();
  }

  // scanline
  const sy2=(tick*1.8)%(ch+40)-20;
  const sg=ctx.createLinearGradient(0,sy2,0,sy2+40);
  sg.addColorStop(0,"rgba(0,255,136,0)");
  sg.addColorStop(.5,"rgba(0,255,136,.025)");
  sg.addColorStop(1,"rgba(0,255,136,0)");
  ctx.fillStyle=sg; ctx.fillRect(0,sy2,cw,40);

  // particles
  particles.forEach(p=>{
    const a=p.life/p.maxLife;
    ctx!.save();
    ctx!.globalAlpha=a;
    ctx!.shadowBlur=8; ctx!.shadowColor=p.color;
    ctx!.beginPath(); ctx!.arc(p.x,p.y,p.size,0,Math.PI*2);
    ctx!.fillStyle=p.color; ctx!.fill();
    ctx!.restore();
  });

  drawHUD();

  // combo popup
  if(comboText&&comboText.life>0){
    const a=comboText.life/45;
    const rise=(45-comboText.life)*.65;
    ctx.save();
    ctx.globalAlpha=a;
    ctx.shadowBlur=15; ctx.shadowColor="#fbd74b";
    ctx.fillStyle="#fbd74b";
    ctx.font=`bold ${14+combo*2}px "Courier New"`;
    ctx.textAlign="center";
    ctx.fillText(comboText.text,comboText.x,comboText.y-rise);
    ctx.textAlign="left";
    ctx.restore();
  }

  ctx.restore();
}

function drawFood(){
  if(!ctx) return;
  if(gameMode.value==="stage" && portal.value) return;
  const px=food.x*CELL+CELL/2, py=food.y*CELL+CELL/2;
  const p=Math.sin(tick*.15)*.28+1;
  ctx.save();
  ctx.shadowBlur=22*p; ctx.shadowColor="#ff0040";
  const g=ctx.createRadialGradient(px,py,0,px,py,CELL/2.4*p);
  g.addColorStop(0,"#ffbbbb"); g.addColorStop(.5,"#ff4060"); g.addColorStop(1,"#cc0030");
  ctx.beginPath(); ctx.arc(px,py,CELL/2.4*p,0,Math.PI*2);
  ctx.fillStyle=g; ctx.fill();
  ctx.restore();
  for(let i=0;i<3;i++){
    const a=tick*.09+(Math.PI*2*i/3);
    ctx.beginPath(); ctx.arc(px+Math.cos(a)*CELL*.62,py+Math.sin(a)*CELL*.62,1.8,0,Math.PI*2);
    ctx.fillStyle="#ff8090"; ctx.fill();
  }
}

function drawPowerUpItem(pu:{x:number;y:number;type:PUType;life:number}){
  if(!ctx) return;
  const px=pu.x*CELL+CELL/2, py=pu.y*CELL+CELL/2;
  const p=Math.sin(tick*.22)*.22+1;
  const col=PU_COLOR[pu.type];
  const r=CELL/2.1*p;
  ctx.save();
  ctx.shadowBlur=16*p; ctx.shadowColor=col;
  ctx.beginPath(); ctx.arc(px,py,r,0,Math.PI*2);
  ctx.fillStyle=col+"28"; ctx.strokeStyle=col; ctx.lineWidth=1.5;
  ctx.fill(); ctx.stroke();
  const ratio=pu.life/22;
  ctx.beginPath(); ctx.arc(px,py,r,-Math.PI/2,-Math.PI/2+Math.PI*2*ratio);
  ctx.strokeStyle=col; ctx.lineWidth=2.5; ctx.stroke();
  ctx.restore();
  ctx.font=`${CELL*.62}px serif`;
  ctx.textAlign="center"; ctx.textBaseline="middle";
  ctx.fillText(PU_ICON[pu.type],px,py);
  ctx.textAlign="left"; ctx.textBaseline="alphabetic";
}

function drawSnake(){
  if(!ctx) return;
  const ghost=activePUs.some(p=>p.type==="ghost");
  const pal = COLOR_PALETTES[isAI.value?aiColor.value:playerColor.value];
  snake.forEach((seg,i)=>{
    const isHead=i===0;
    const t=i/(snake.length-1||1);
    const r=Math.round(lerp(pal.head[0],pal.tail[0],t));
    const g=Math.round(lerp(pal.head[1],pal.tail[1],t));
    const b=Math.round(lerp(pal.head[2],pal.tail[2],t));
    const alpha=ghost ? 0.55 : 1;
    const px=seg.x*CELL+CELL/2, py=seg.y*CELL+CELL/2;
    const rad=Math.max(3,(isHead?CELL/2.1:CELL/2.4-t*2));
    ctx!.save();
    ctx!.shadowBlur=isHead?20:10-t*8;
    ctx!.shadowColor=`rgb(${r},${g},${b})`;
    ctx!.globalAlpha=alpha;
    ctx!.beginPath(); ctx!.arc(px,py,rad,0,Math.PI*2);
    ctx!.fillStyle=`rgb(${r},${g},${b})`; ctx!.fill();
    ctx!.restore();

    if(isHead){
      const eo=CELL*.27,er=CELL*.1;
      const dx=dir==="LEFT"?-1:dir==="RIGHT"?1:0;
      const dy=dir==="UP"?-1:dir==="DOWN"?1:0;
      const px2=-dy,py2=dx;
      const e1x=px+dx*eo+px2*eo*.65,e1y=py+dy*eo+py2*eo*.65;
      const e2x=px+dx*eo-px2*eo*.65,e2y=py+dy*eo-py2*eo*.65;
      ctx!.globalAlpha=alpha;
      ctx!.fillStyle="#000";
      ctx!.beginPath();ctx!.arc(e1x,e1y,er,0,Math.PI*2);ctx!.fill();
      ctx!.beginPath();ctx!.arc(e2x,e2y,er,0,Math.PI*2);ctx!.fill();
      ctx!.fillStyle="#fff";
      ctx!.beginPath();ctx!.arc(e1x-er*.3,e1y-er*.3,er*.35,0,Math.PI*2);ctx!.fill();
      ctx!.beginPath();ctx!.arc(e2x-er*.3,e2y-er*.3,er*.35,0,Math.PI*2);ctx!.fill();
    }
  });
}

// ── draw duel (雙人對戰) ──────────────────────────────────
function drawDuelGrid(g:1|2, ox:number, oy:number){
  if(!ctx) return;
  const c=ctx;
  const isG1=g===1;
  const myFood=isG1?food:food2;
  const myTraps=isG1?traps1:traps2;
  const myPortal=isG1?portal1:portal2;
  const myPU=isG1?fieldPU:fieldPU2;

  // grid background（加粗格線）
  c.strokeStyle=isG1?"rgba(0,255,170,.12)":"rgba(192,132,252,.12)";
  c.lineWidth=1;
  for(let i=0;i<=GRID;i++){
    c.beginPath();c.moveTo(ox+i*DUEL_CW,oy);c.lineTo(ox+i*DUEL_CW,oy+DUEL_GH);c.stroke();
    c.beginPath();c.moveTo(ox,oy+i*DUEL_CH);c.lineTo(ox+DUEL_GW,oy+i*DUEL_CH);c.stroke();
  }

  // grid border glow（加亮）
  c.save();
  c.shadowBlur=30;
  c.shadowColor=isG1?"#00ffaa":"#c084fc";
  c.strokeStyle=isG1?"rgba(0,255,170,.6)":"rgba(192,132,252,.6)";
  c.lineWidth=4;
  c.strokeRect(ox,oy,DUEL_GW,DUEL_GH);
  c.restore();

  // food
  if(myFood){
    const px=ox+myFood.x*DUEL_CW+DUEL_CW/2, py=oy+myFood.y*DUEL_CH+DUEL_CH/2;
    const p=Math.sin(tick*.15)*.28+1;
    c.save();
    c.shadowBlur=22*p; c.shadowColor="#ff0040";
    const gr=c.createRadialGradient(px,py,0,px,py,DUEL_CW/2.4*p);
    gr.addColorStop(0,"#ffbbbb"); gr.addColorStop(.5,"#ff4060"); gr.addColorStop(1,"#cc0030");
    c.beginPath();c.arc(px,py,DUEL_CW/2.4*p,0,Math.PI*2);
    c.fillStyle=gr; c.fill();
    c.restore();
  }

  // traps
  for(const t of myTraps){
    const pulse=0.7+Math.sin(tick*0.2+t.x+t.y)*0.3;
    c.save();
    c.shadowBlur=8; c.shadowColor="#f87171";
    c.fillStyle="#f87171"; c.globalAlpha=0.5*pulse;
    c.fillRect(ox+t.x*DUEL_CW+1,oy+t.y*DUEL_CH+1,DUEL_CW-2,DUEL_CH-2);
    c.restore();
    c.save();
    c.strokeStyle="#f87171"; c.globalAlpha=0.7*pulse; c.lineWidth=1.5;
    c.beginPath();c.moveTo(ox+t.x*DUEL_CW+4,oy+t.y*DUEL_CH+4);c.lineTo(ox+t.x*DUEL_CW+DUEL_CW-4,oy+t.y*DUEL_CH+DUEL_CH-4);c.stroke();
    c.beginPath();c.moveTo(ox+t.x*DUEL_CW+DUEL_CW-4,oy+t.y*DUEL_CH+4);c.lineTo(ox+t.x*DUEL_CW+4,oy+t.y*DUEL_CH+DUEL_CH-4);c.stroke();
    c.restore();
  }

  // portal
  if(myPortal){
    const px=ox+myPortal.x*DUEL_CW+DUEL_CW/2, py=oy+myPortal.y*DUEL_CH+DUEL_CH/2;
    const r=DUEL_CW/2.2+Math.sin(tick*0.15)*3;
    c.save();
    c.shadowBlur=20; c.shadowColor="#7c3aed";
    const grad=c.createRadialGradient(px,py,0,px,py,r);
    grad.addColorStop(0,"rgba(124,58,237,0.9)");
    grad.addColorStop(0.5,"rgba(139,92,246,0.5)");
    grad.addColorStop(1,"rgba(124,58,237,0)");
    c.beginPath();c.arc(px,py,r,0,Math.PI*2);
    c.fillStyle=grad; c.fill();
    c.strokeStyle="#a78bfa"; c.lineWidth=1.5;
    c.globalAlpha=0.6+Math.sin(tick*0.2)*0.3;
    c.beginPath();c.arc(px,py,r+2,0,Math.PI*1.5+tick*0.05);c.stroke();
    c.restore();
    c.save();
    c.font="14px serif"; c.textAlign="center"; c.textBaseline="middle";
    c.fillText("🌀",px,py);
    c.textAlign="left"; c.textBaseline="alphabetic";
    c.restore();
  }

  // power-up item
  if(myPU){
    const px=ox+myPU.x*DUEL_CW+DUEL_CW/2, py=oy+myPU.y*DUEL_CH+DUEL_CH/2;
    const p2=Math.sin(tick*.22)*.22+1;
    const col=PU_COLOR[myPU.type];
    const rad=DUEL_CW/2.1*p2;
    c.save();
    c.shadowBlur=14*p2; c.shadowColor=col;
    c.beginPath();c.arc(px,py,rad,0,Math.PI*2);
    c.fillStyle=col+"28"; c.strokeStyle=col; c.lineWidth=1.2;
    c.fill();c.stroke();
    const ratio=myPU.life/22;
    c.beginPath();c.arc(px,py,rad,-Math.PI/2,-Math.PI/2+Math.PI*2*ratio);
    c.strokeStyle=col; c.lineWidth=2; c.stroke();
    c.restore();
    c.font=`${DUEL_CW*.55}px serif`;
    c.textAlign="center";c.textBaseline="middle";
    c.fillText(PU_ICON[myPU.type],px,py);
    c.textAlign="left";c.textBaseline="alphabetic";
  }
}

function drawSnakeAt(snake:{x:number;y:number}[], dir:string, ox:number, oy:number, ghost:boolean, pal:ColorPalette){
  if(!ctx) return;
  const c=ctx;
  snake.forEach((seg,i)=>{
    const isHead=i===0;
    const t=i/(snake.length-1||1);
    const r=Math.round(lerp(pal.head[0],pal.tail[0],t));
    const g=Math.round(lerp(pal.head[1],pal.tail[1],t));
    const b=Math.round(lerp(pal.head[2],pal.tail[2],t));
    const alpha=ghost?0.55:1;
    const px=ox+seg.x*DUEL_CW+DUEL_CW/2, py=oy+seg.y*DUEL_CH+DUEL_CH/2;
    const rad=Math.max(2.5,(isHead?DUEL_CW/2.1:DUEL_CW/2.4-t*1.5));
    c.save();
    c.shadowBlur=isHead?16:8-t*6;
    c.shadowColor=`rgb(${r},${g},${b})`;
    c.globalAlpha=alpha;
    c.beginPath();c.arc(px,py,rad,0,Math.PI*2);
    c.fillStyle=`rgb(${r},${g},${b})`;c.fill();
    c.restore();
    if(isHead){
      const eo=DUEL_CW*.27,er=DUEL_CW*.1;
      const dx=dir==="LEFT"?-1:dir==="RIGHT"?1:0;
      const dy=dir==="UP"?-1:dir==="DOWN"?1:0;
      const px2=-dy,py2=dx;
      c.save();
      c.globalAlpha=alpha;
      c.fillStyle="#000";
      c.beginPath();c.arc(px+dx*eo+px2*eo*.65,py+dy*eo+py2*eo*.65,er,0,Math.PI*2);c.fill();
      c.beginPath();c.arc(px+dx*eo-px2*eo*.65,py+dy*eo-py2*eo*.65,er,0,Math.PI*2);c.fill();
      c.fillStyle="#fff";
      c.beginPath();c.arc(px+dx*eo+px2*eo*.65-er*.3,py+dy*eo+py2*eo*.65-er*.3,er*.35,0,Math.PI*2);c.fill();
      c.beginPath();c.arc(px+dx*eo-px2*eo*.65-er*.3,py+dy*eo-py2*eo*.65-er*.3,er*.35,0,Math.PI*2);c.fill();
      c.restore();
    }
  });
}

function drawDuel(){
  if(!ctx) return;
  const c=ctx;

  const oy=0;

  // draw grid 1 (left half: P1)
  drawDuelGrid(1,0,oy);
  // draw grid 2 (right half: P2)
  drawDuelGrid(2,DUEL_GW,oy);

  // divider line
  c.strokeStyle="rgba(255,255,255,.06)";
  c.lineWidth=1;
  c.beginPath();c.moveTo(DUEL_GW-0.5,oy);c.lineTo(DUEL_GW-0.5,oy+DUEL_GH);c.stroke();

  // label grids
  c.fillStyle="rgba(0,255,170,.2)";
  c.font='bold 10px "Courier New"';
  c.textAlign="center";
  c.fillText("——— P1 ———",DUEL_GW/2,oy+12);
  c.fillStyle="rgba(192,132,252,.2)";
  c.fillText("——— P2 ———",DUEL_GW+DUEL_GW/2,oy+12);
  c.textAlign="left";

  // draw snakes in correct grids
  const pal1=COLOR_PALETTES[playerColor.value];
  const pal2=COLOR_PALETTES[is2PAI.value?aiColor.value:"purple"];
  const g1=getCurrentGrid(1);
  const g2=getCurrentGrid(2);
  const ox1=g1===1?0:DUEL_GW;
  const ox2=g2===1?0:DUEL_GW;
  const ghost1=activePUs.some(p=>p.type==="ghost");
  const ghost2=activePUs2.some(p=>p.type==="ghost");

  drawSnakeAt(snake,dir,ox1,oy,ghost1,pal1);
  drawSnakeAt(snake2,dir2,ox2,oy,ghost2,pal2);
}

// ── draw boss ─────────────────────────────────────────────
function drawBoss(){
  if(!ctx||!bossData) return;
  const c=ctx; const b=bossData;
  const alert=b.alert>0;
  b.body.forEach((seg,i)=>{
    const isHead=i===0;
    const t=i/(b.body.length-1||1);
    const pulse=alert?1+Math.sin(tick*.5)*.3:1;
    const rad=isHead?CELL/1.8*pulse:Math.max(5,(CELL/2.2-t*1.5)*pulse);
    const px=seg.x*CELL+CELL/2, py=seg.y*CELL+CELL/2;
    const col=alert?`hsl(0,100%,${50+Math.sin(tick*.3)*20}%)`:`hsl(${340-t*40},100%,${55-t*15}%)`;
    c.save();
    c.shadowBlur=isHead?35:15;
    c.shadowColor=alert?"#ff0000":"#ff4060";
    c.globalAlpha=alert ? 0.8+t*0.2 : 1;
    c.beginPath(); c.arc(px,py,rad,0,Math.PI*2);
    c.fillStyle=col; c.fill();
    c.restore();
    if(isHead&&!alert){
      const eo=CELL*.35,er=CELL*.16;
      const dx=b.dir==="LEFT"?-1:b.dir==="RIGHT"?1:0;
      const dy=b.dir==="UP"?-1:b.dir==="DOWN"?1:0;
      const px2=-dy,py2=dx;
      c.fillStyle="#200";
      c.beginPath();c.arc(px+dx*eo+px2*eo*.65,py+dy*eo+py2*eo*.65,er,0,Math.PI*2);c.fill();
      c.beginPath();c.arc(px+dx*eo-px2*eo*.65,py+dy*eo-py2*eo*.65,er,0,Math.PI*2);c.fill();
      c.fillStyle="#ff6060";
      c.beginPath();c.arc(px+dx*eo+px2*eo*.65-er*.3,py+dy*eo+py2*eo*.65-er*.3,er*.4,0,Math.PI*2);c.fill();
      c.beginPath();c.arc(px+dx*eo-px2*eo*.65-er*.3,py+dy*eo-py2*eo*.65-er*.3,er*.4,0,Math.PI*2);c.fill();
    }
  });
}

// ── draw missiles ─────────────────────────────────────────
function drawMissiles(){
  if(!ctx) return;
  const c=ctx;
  for(const m of missiles){
    const px=m.x*CELL+CELL/2, py=m.y*CELL+CELL/2;
    const a=Math.atan2(m.dy,m.dx);
    c.save();
    c.translate(px,py);
    c.rotate(a);
    c.shadowBlur=15; c.shadowColor="#fbd74b";
    c.fillStyle="#fbd74b";
    c.beginPath();
    c.moveTo(8,0); c.lineTo(-5,-4); c.lineTo(-3,0); c.lineTo(-5,4);
    c.closePath(); c.fill();
    c.restore();
  }
}

function drawStage(){
  if(!ctx) return;
  const c=ctx;
  // traps (hide when portal is open)
  if(!portal.value) for(const t of traps){
    const pulse=0.7+Math.sin(tick*0.2+t.x+t.y)*0.3;
    c.save();
    c.shadowBlur=10; c.shadowColor="#f87171";
    c.fillStyle="#f87171";
    c.globalAlpha=0.6*pulse;
    c.fillRect(t.x*CELL+2,t.y*CELL+2,CELL-4,CELL-4);
    c.restore();
    // X mark
    c.save();
    c.strokeStyle="#f87171";
    c.globalAlpha=0.8*pulse;
    c.lineWidth=2;
    c.beginPath(); c.moveTo(t.x*CELL+5,t.y*CELL+5); c.lineTo(t.x*CELL+CELL-5,t.y*CELL+CELL-5); c.stroke();
    c.beginPath(); c.moveTo(t.x*CELL+CELL-5,t.y*CELL+5); c.lineTo(t.x*CELL+5,t.y*CELL+CELL-5); c.stroke();
    c.restore();
  }
  // portal
  if(portal.value){
    const p=portal.value;
    const px=p.x*CELL+CELL/2, py=p.y*CELL+CELL/2;
    const r=CELL/2.2+Math.sin(tick*0.15)*4;
    c.save();
    c.shadowBlur=25; c.shadowColor="#7c3aed";
    const grad=c.createRadialGradient(px,py,0,px,py,r);
    grad.addColorStop(0,"rgba(124,58,237,0.9)");
    grad.addColorStop(0.5,"rgba(139,92,246,0.5)");
    grad.addColorStop(1,"rgba(124,58,237,0)");
    c.beginPath(); c.arc(px,py,r,0,Math.PI*2);
    c.fillStyle=grad; c.fill();
    // swirl ring
    c.strokeStyle="#a78bfa";
    c.lineWidth=2;
    c.globalAlpha=0.6+Math.sin(tick*0.2)*0.3;
    c.beginPath(); c.arc(px,py,r+3,0,Math.PI*1.5+tick*0.05); c.stroke();
    c.restore();
    c.save();
    c.font="18px serif";
    c.textAlign="center"; c.textBaseline="middle";
    c.fillText("🌀",px,py);
    c.textAlign="left"; c.textBaseline="alphabetic";
    c.restore();
  }
}

function drawHUD(){
  if(!ctx) return;
  // score (hidden in duel mode—shown in custom HUD)
  if(gameMode.value!=="tron"){
    ctx.save();
    ctx.shadowBlur=12; ctx.shadowColor="#00ffaa";
    ctx.fillStyle="#00ffaa";
    ctx.font='bold 18px "Courier New"';
    ctx.fillText(`${score.value} pts`,10,24);
    ctx.restore();
  }

  // mode label
  if(gameMode.value==="tron"){
    // duel HUD — full-width layout
    const dOx=0;
    const dHudY=DUEL_GH+4;
    // P1 info (left)
    ctx.fillStyle="#00ffaa";
    ctx.font='bold 11px "Courier New"';
    ctx.fillText(`P1 ${playerName.value||""} 🟢`,dOx+4,dHudY);
    ctx.font='10px "Courier New"';
    ctx.fillText(`命 ${"❤️".repeat(lives1)} 分 ${score1}`,dOx+4,dHudY+14);
    if(heldSkill1){
      ctx.fillStyle=PU_COLOR[heldSkill1];
      ctx.fillText(`[SPACE] ${PU_LABEL[heldSkill1]}`,dOx+4,dHudY+28);
    }
    // P2 info (right)
    ctx.fillStyle="#c084fc";
    ctx.font='bold 11px "Courier New"';
    ctx.textAlign="right";
    ctx.fillText(`🟣 P2 ${is2PAI.value?"🤖 AI":(playerName2.value||"")}`,SIZE*2-4,dHudY);
    ctx.font='10px "Courier New"';
    ctx.fillText(`命 ${"❤️".repeat(lives2)} 分 ${score2}`,SIZE*2-4,dHudY+14);
    if(heldSkill2){
      ctx.fillStyle=PU_COLOR[heldSkill2];
      ctx.fillText(`[0] ${PU_LABEL[heldSkill2]}`,SIZE*2-4,dHudY+28);
    }
    ctx.textAlign="left";
    // active PUs — P1 on left, P2 on right
    activePUs.forEach((pu,i)=>{
      const bw=52,x=4+i*(bw+6),y=dHudY+40;
      const ratio=pu.steps/pu.maxSteps;
      const col=PU_COLOR[pu.type];
      ctx!.fillStyle="rgba(0,0,0,.5)"; ctx!.fillRect(x,y,bw,13);
      ctx!.fillStyle=col+"55"; ctx!.fillRect(x,y,bw*ratio,13);
      ctx!.strokeStyle=col; ctx!.lineWidth=1; ctx!.strokeRect(x,y,bw,13);
      ctx!.fillStyle=col;
      ctx!.font='bold 8px "Courier New"';
      ctx!.textAlign="center"; ctx!.fillText(PU_LABEL[pu.type],x+bw/2,y+10); ctx!.textAlign="left";
    });
    activePUs2.forEach((pu,i)=>{
      const bw=52,x=SIZE*2-4-(i+1)*(bw+6),y=dHudY+40;
      const ratio=pu.steps/pu.maxSteps;
      const col=PU_COLOR[pu.type];
      ctx!.fillStyle="rgba(0,0,0,.5)"; ctx!.fillRect(x,y,bw,13);
      ctx!.fillStyle=col+"55"; ctx!.fillRect(x,y,bw*ratio,13);
      ctx!.strokeStyle=col; ctx!.lineWidth=1; ctx!.strokeRect(x,y,bw,13);
      ctx!.fillStyle=col;
      ctx!.font='bold 8px "Courier New"';
      ctx!.textAlign="center"; ctx!.fillText(PU_LABEL[pu.type],x+bw/2,y+10); ctx!.textAlign="left";
    });
  } else if(gameMode.value==="boss"){
    ctx.fillStyle="#ff4060";
    ctx.font='bold 11px "Courier New"';
    ctx.fillText(`👑 BOSS MODE`,10,40);
    ctx.fillStyle="rgba(255,255,255,.38)";
    ctx.font='11px "Courier New"';
    ctx.fillText(`LV ${level}`,10,56);
  } else if(gameMode.value==="stage"){
    ctx.fillStyle="#a78bfa";
    ctx.font='bold 11px "Courier New"';
    ctx.fillText(`🏰 STAGE ${stage.value}/${MAX_STAGE}`,10,40);
    ctx.fillStyle="rgba(255,255,255,.38)";
    ctx.font='11px "Courier New"';
    const target=getStageTarget(stage.value);
    ctx.fillText(`目標 ${target} pts (${score.value}/${target})`,10,56);
  } else {
    ctx.fillStyle="rgba(255,255,255,.38)";
    ctx.font='11px "Courier New"';
    ctx.fillText(`LV ${level}`,10,40);
  }

  ctx.textAlign="right";
  const showP2 = gameMode.value==="tron";
  const p1Label = isAI.value ? "🤖 AI" : (playerName.value.trim() || "P1");
  const p2Label = is2PAI.value ? "🤖 AI" : (playerName2.value.trim() || "P2");
  if(gameMode.value!=="tron"){
    ctx.fillStyle="#00ffaa";
    ctx.font='bold 11px "Courier New"';
    ctx.fillText(p1Label,SIZE-8,24);
  }
  if(showP2 && gameMode.value!=="tron"){
    ctx.fillStyle="#c084fc";
    ctx.font='bold 11px "Courier New"';
    ctx.fillText("🟣 "+p2Label,SIZE-8,40);
  }
  ctx.textAlign="left";

  // boss HP bar
  if(gameMode.value==="boss"&&bossData?.active){
    const bw=120, bh=10, x=SIZE/2-bw/2, y=8;
    const ratio=Math.max(0,bossData.health/bossData.maxHealth);
    const col=bossData.health>10?"#ff4060":"#ff0000";
    ctx.save();
    ctx.shadowBlur=12; ctx.shadowColor="#ff0040";
    ctx.fillStyle="rgba(0,0,0,.6)"; ctx.fillRect(x,y,bw,bh);
    ctx.fillStyle=col+"88"; ctx.fillRect(x+1,y+1,(bw-2)*ratio,bh-2);
    ctx.strokeStyle=col; ctx.lineWidth=1; ctx.strokeRect(x,y,bw,bh);
    ctx.fillStyle="#fff";
    ctx.font='bold 8px "Courier New"';
    ctx.textAlign="center";
    ctx.fillText(`BOSS  ${bossData.health}/${bossData.maxHealth}`,SIZE/2,y+bh-1);
    ctx.textAlign="left";
    ctx.restore();
    // warning pulse
    if(bossData.health<=5){
      ctx.save();
      ctx.globalAlpha=.3+Math.sin(tick*.15)*.2;
      ctx.fillStyle="#ff0000";
      ctx.fillRect(0,0,SIZE,SIZE);
      ctx.restore();
    }
  }

  // missile count (boss mode)
  if(gameMode.value==="boss"&&bossData?.active){
    ctx.fillStyle="#fbd74b";
    ctx.font='bold 10px "Courier New"';
    ctx.fillText(`🚀 x${5-missiles.length}  [SPACE]`,10,SIZE-8);
  }

  // 手持技能提示（非對戰模式）
  if(gameMode.value!=="tron"&&heldPU.value){
    const col=PU_COLOR[heldPU.value];
    ctx.save();
    ctx.fillStyle=col;
    ctx.font='bold 10px "Courier New"';
    ctx.textAlign="center";
    ctx.fillText(`[SPACE] ${PU_ICON[heldPU.value]} ${PU_LABEL[heldPU.value]}`,SIZE/2,SIZE-28);
    ctx.textAlign="left";
    ctx.restore();
  }

  // active PUs (non-duel mode)
  if(gameMode.value!=="tron") activePUs.forEach((pu,i)=>{
    const bw=68,x=10+i*(bw+8),y=SIZE-22;
    const ratio=pu.steps/pu.maxSteps;
    const col=PU_COLOR[pu.type];
    ctx!.fillStyle="rgba(0,0,0,.6)"; ctx!.fillRect(x,y,bw,16);
    ctx!.fillStyle=col+"55"; ctx!.fillRect(x,y,bw*ratio,16);
    ctx!.strokeStyle=col; ctx!.lineWidth=1; ctx!.strokeRect(x,y,bw,16);
    ctx!.fillStyle=col;
    ctx!.font='bold 9px "Courier New"';
    ctx!.textAlign="center"; ctx!.fillText(PU_LABEL[pu.type],x+bw/2,y+11); ctx!.textAlign="left";
  });
}

onMounted(()=>{document.addEventListener("keydown",key);document.addEventListener("keyup",keyup);loadLB("normal");loadHist("normal");loadShop();});
onUnmounted(()=>{document.removeEventListener("keydown",key);document.removeEventListener("keyup",keyup);if(replayTimer)clearTimeout(replayTimer);});
</script>

<template>
  <div class="wrap">

    <!-- ===== MENU ===== -->
    <div v-if="state==='menu'" class="layout">

      <!-- ─── MAIN: 3 mode cards ─── -->
      <template v-if="selectedMode==='main'">
        <div class="center main-center">
          <div class="logo">
            <span class="logo-txt">NEON</span>
            <span class="logo-snake">🐍</span>
            <span class="logo-txt">SNAKE</span>
          </div>
          <div class="tagline">— CHOOSE YOUR MODE —</div>
          <div class="mode-cards">
            <div class="mode-card" @click="selectMode('normal')">
              <span class="mc-icon">🐍</span>
              <span class="mc-title">經典模式</span>
              <span class="mc-desc">吃食物不斷成長，挑戰最高分數</span>
            </div>
            <div class="mode-card" @click="selectMode('tron')">
              <span class="mc-icon">⚡</span>
              <span class="mc-title">雙人對戰</span>
              <span class="mc-desc">雙蛇對戰，各自為陣！穿越傳送門入侵對手</span>
            </div>
            <div class="mode-card" @click="selectMode('boss')">
              <span class="mc-icon">👑</span>
              <span class="mc-title">Boss 戰</span>
              <span class="mc-desc">擊敗巨型 Boss 賺取高分</span>
            </div>
            <div class="mode-card" @click="selectMode('stage')">
              <span class="mc-icon">🏰</span>
              <span class="mc-title">關卡遊戲</span>
              <span class="mc-desc">連續挑戰 20 關，穿越黑洞晉級</span>
            </div>
          </div>
          <div class="color-picker">
            <div class="cp-label">🐍 蛇的顏色</div>
            <div class="cp-row">
              <span v-for="k in colorKeys" :key="k"
                class="cp-swatch" :class="{active:playerColor===k}"
                :style="{background:`rgb(${COLOR_PALETTES[k].head[0]},${COLOR_PALETTES[k].head[1]},${COLOR_PALETTES[k].head[2]})`}"
                @click="playerColor=k">
              </span>
            </div>
          </div>
          <div class="shop-bar">
            <span class="shop-coins">🪙 {{coins}}</span>
            <button class="btn-shop" @click="selectMode('shop')">🏪 商店</button>
          </div>
        </div>
      </template>

      <!-- ─── MODE DETAIL ─── -->
      <template v-else>
        <div class="sidebar" v-if="selectedMode!=='tron'&&selectedMode!=='shop'">
          <div class="lb-box">
            <div class="lb-title">🏆 排行榜 <span class="lb-mode">{{selectedMode==='normal'?'經典':selectedMode==='boss'?'Boss戰':'關卡'}}</span></div>
            <div v-if="leaderboard.length===0" class="lb-empty">尚無紀錄</div>
            <div v-for="(e,i) in leaderboard" :key="i" class="lb-row" :class="'r'+(i+1)">
              <span class="lb-medal">{{i===0?'🥇':i===1?'🥈':i===2?'🥉':'#'+(i+1)}}</span>
              <span class="lb-name">{{e.name}}</span>
              <span class="lb-score">{{e.score}}</span>
            </div>
          </div>
          <div class="hist-box" v-if="history.length>0">
            <div class="hist-title">📼 回放紀錄</div>
            <div class="hist-scroll">
              <div v-for="(h,i) in history" :key="i" class="hist-card" @click="playReplay(h)">
                <span class="hc-idx">第{{i+1}}局</span>
                <span class="hc-who">{{h.label}}</span>
                <span class="hc-score">{{h.score}}分</span>
              </div>
            </div>
          </div>
        </div>

        <div class="center">
          <button class="back-btn" @click="selectMode('main')">← 返回選單</button>

          <!-- Normal mode detail -->
          <template v-if="selectedMode==='normal'">
            <div class="mode-hero">
              <span class="mh-icon">🐍</span>
              <div class="mh-title">經典模式</div>
              <div class="mh-desc">控制蛇在網格中移動，吃掉紅色食物來增長身體與得分。<br/>小心不要撞到牆壁或自己的尾巴！沿途收集五種能力道具扭轉局勢。</div>
            </div>
            <div class="powers-row">
              <span v-for="t in (['shield','ghost','double','shrink','slow'] as PUType[])" :key="t"
                class="pp" :class="{active: selectedPU===t}"
                :style="{color:puColor(t), filter:`drop-shadow(0 0 6px ${puColor(t)})`}"
                @click="togglePUInfo(t)">{{puIcon(t)}}</span>
            </div>
            <transition name="pu-fade">
              <div v-if="selectedPU" class="pu-tooltip"
                :style="{borderColor: puColor(selectedPU), boxShadow:`0 0 18px ${puColor(selectedPU)}44`}">
                <span class="pu-tt-icon">{{puIcon(selectedPU)}}</span>
                <div class="pu-tt-body">
                  <div class="pu-tt-name" :style="{color: puColor(selectedPU)}">{{ PU_LABEL[selectedPU] }}</div>
                  <div class="pu-tt-desc">{{ PU_DESC[selectedPU] }}</div>
                </div>
                <button class="pu-tt-close" @click.stop="selectedPU=null">✕</button>
              </div>
            </transition>
            <div class="name-wrap">
              <label class="name-lbl">玩家名稱</label>
              <input v-model="playerName" class="name-inp" :class="{err:nameError}"
                placeholder="輸入你的名稱..." maxlength="12" @keyup.enter="startGame('normal', false)" />
              <span v-if="nameError" class="name-err">請輸入名稱！</span>
            </div>
            <div class="btn-row">
              <button class="btn-play" @click="startGame('normal', false)">▶ 自己玩</button>
              <button class="btn-ai" @click="startGame('normal', true)">🤖 看 AI 玩</button>
            </div>
          </template>

          <!-- Tron mode detail -->
          <template v-if="selectedMode==='tron'">
            <div class="mode-hero">
              <span class="mh-icon">⚡</span>
              <div class="mh-title">⚡ 雙人對戰</div>
              <div class="mh-desc">
                兩位玩家在各自的 20×20 戰場對決！<br/>
                <b>穿越傳送門</b>入侵對方領地，擊殺對手偷取一條命！<br/>
                每人 3 條命，率先耗盡對方生命者獲勝！<br/>
                按 <b>空白鍵 (P1)</b> 或 <b>0 鍵 (P2)</b> 發動持有的能力道具！
              </div>
            </div>
            <div class="section-title">⚡ 模式特色</div>
            <div class="powers-row">
              <span v-for="item in MODE_ITEMS['tron']" :key="item.id"
                class="pp" :class="{active: selectedModeItem===item.id}"
                style="color:#c084fc;filter:drop-shadow(0 0 6px #c084fc)"
                @click="toggleModeItem(item.id)">{{item.icon}}</span>
            </div>
            <transition name="pu-fade">
              <div v-if="selectedModeItem" class="pu-tooltip"
                :style="{borderColor:'#c084fc',boxShadow:'0 0 18px #c084fc44'}">
                <span class="pu-tt-icon">{{modeItem('tron')?.icon}}</span>
                <div class="pu-tt-body">
                  <div class="pu-tt-name" style="color:#c084fc">{{modeItem('tron')?.label}}</div>
                  <div class="pu-tt-desc">{{modeItem('tron')?.desc}}</div>
                </div>
                <button class="pu-tt-close" @click.stop="selectedModeItem=null">✕</button>
              </div>
            </transition>
            <div class="section-divider">🎮 開始對戰</div>
            <div class="name-row">
              <div class="name-wrap" style="flex:1;">
                <label class="name-lbl">P1 名稱</label>
                <input v-model="playerName" class="name-inp" placeholder="P1..." maxlength="12"
                  @keyup.enter="startGame('tron', false)" />
              </div>
              <div class="name-wrap" style="flex:1;">
                <label class="name-lbl">P2 名稱</label>
                <input v-model="playerName2" class="name-inp" placeholder="P2..." maxlength="12"
                  @keyup.enter="startGame('tron', false)" />
              </div>
            </div>
            <div class="btn-row">
              <button class="btn-play" @click="startGame('tron', false)">🎮 WASD 對戰</button>
              <button class="btn-ai" @click="startGame('tron', true)">🤖 vs AI 電腦</button>
            </div>
            <div class="ctrl-hint">
              <span>P1 <b>W A S D</b> + <b>空白鍵</b>(技能)</span> ｜
              <span>P2 ⬆⬇⬅➡ + <b>0</b>(技能)</span>
            </div>
          </template>

          <!-- Boss mode detail -->
          <template v-if="selectedMode==='boss'">
            <div class="mode-hero">
              <span class="mh-icon">👑</span>
              <div class="mh-title">Boss 戰</div>
              <div class="mh-desc">經典玩法升級！當分數達到 15 分時，巨型 Boss 蛇會現身追殺你。<br/>
                收集飛彈按 <b>空白鍵 [SPACE]</b> 射擊 Boss，擊敗可獲得大量加分獎勵！</div>
            </div>
            <div class="section-title">⚡ 模式特色</div>
            <div class="powers-row">
              <span v-for="item in MODE_ITEMS['boss']" :key="item.id"
                class="pp" :class="{active: selectedModeItem===item.id}"
                style="color:#fbd74b;filter:drop-shadow(0 0 6px #fbd74b)"
                @click="toggleModeItem(item.id)">{{item.icon}}</span>
            </div>
            <transition name="pu-fade">
              <div v-if="selectedModeItem" class="pu-tooltip"
                :style="{borderColor:'#fbd74b',boxShadow:'0 0 18px #fbd74b44'}">
                <span class="pu-tt-icon">{{modeItem('boss')?.icon}}</span>
                <div class="pu-tt-body">
                  <div class="pu-tt-name" style="color:#fbd74b">{{modeItem('boss')?.label}}</div>
                  <div class="pu-tt-desc">{{modeItem('boss')?.desc}}</div>
                </div>
                <button class="pu-tt-close" @click.stop="selectedModeItem=null">✕</button>
              </div>
            </transition>
            <div class="name-wrap">
              <label class="name-lbl">玩家名稱</label>
              <input v-model="playerName" class="name-inp" :class="{err:nameError}"
                placeholder="輸入你的名稱..." maxlength="12" @keyup.enter="startGame('boss', false)" />
              <span v-if="nameError" class="name-err">請輸入名稱！</span>
            </div>
            <div class="btn-row">
              <button class="btn-play" @click="startGame('boss', false)">▶ 開始挑戰</button>
              <button class="btn-ai" @click="startGame('boss', true)">🤖 看 AI 玩</button>
            </div>
          </template>

          <!-- Stage mode detail -->
          <template v-if="selectedMode==='stage'">
            <div class="mode-hero">
              <span class="mh-icon">🏰</span>
              <div class="mh-title">關卡遊戲 — 穿越黑洞</div>
              <div class="mh-desc">
                共 <b>20 關</b>，每關需達到指定分數<br/>
                到達分數後 <b>紫色黑洞</b> 會出現，蛇頭鑽進去就晉級！<br/>
                後面關卡會有 <b>紅色陷阱</b> 擋路，速度越來越快<br/>
                第 20 關幾乎不可能完成，你能撐多遠？<br/>
                每過一關獲得 🪙 獎勵金幣，可在商店購買道具！
              </div>
            </div>
            <div class="section-title">⚡ 模式特色</div>
            <div class="powers-row">
              <span v-for="item in MODE_ITEMS['stage']" :key="item.id"
                class="pp" :class="{active: selectedModeItem===item.id}"
                style="color:#a78bfa;filter:drop-shadow(0 0 6px #a78bfa)"
                @click="toggleModeItem(item.id)">{{item.icon}}</span>
            </div>
            <transition name="pu-fade">
              <div v-if="selectedModeItem" class="pu-tooltip"
                :style="{borderColor:'#a78bfa',boxShadow:'0 0 18px #a78bfa44'}">
                <span class="pu-tt-icon">{{modeItem('stage')?.icon}}</span>
                <div class="pu-tt-body">
                  <div class="pu-tt-name" style="color:#a78bfa">{{modeItem('stage')?.label}}</div>
                  <div class="pu-tt-desc">{{modeItem('stage')?.desc}}</div>
                </div>
                <button class="pu-tt-close" @click.stop="selectedModeItem=null">✕</button>
              </div>
            </transition>
            <div class="name-wrap">
              <label class="name-lbl">玩家名稱</label>
              <input v-model="playerName" class="name-inp" :class="{err:nameError}"
                placeholder="輸入你的名稱..." maxlength="12" @keyup.enter="startGame('stage', false)" />
            </div>
            <div class="btn-row">
              <button class="btn-play" @click="startGame('stage', false)">🏰 開始闖關</button>
            </div>
          </template>

          <!-- Shop page -->
          <template v-if="selectedMode==='shop'">
            <div class="mode-hero">
              <span class="mh-icon">🏪</span>
              <div class="mh-title">商店</div>
              <div class="mh-desc">使用闖關賺來的金幣購買道具，<br/>所有模式都適用！</div>
            </div>
            <div class="shop-coins-display">🪙 {{coins}} 金幣</div>
            <div class="shop-items">
              <div v-for="item in SHOP_ITEMS" :key="item.id" class="shop-item"
                :class="{owned:ownedItems.has(item.id)}">
                <span class="si-icon">{{item.icon}}</span>
                <div class="si-body">
                  <div class="si-name">{{item.name}}</div>
                  <div class="si-desc">{{item.desc}}</div>
                </div>
                <button v-if="!ownedItems.has(item.id)" class="btn-buy"
                  :class="{cant:coins<item.cost}" @click="buyItem(item.id)">
                  🪙 {{item.cost}}
                </button>
                <span v-else class="si-owned">✔ 已擁有</span>
              </div>
            </div>
          </template>
        </div>
      </template>
    </div>

    <!-- ===== GAME OVER ===== -->
    <div v-else-if="state==='gameover'" class="layout">
      <div class="center">
        <div class="go-title">{{gameMode==='tron'?'⚡ GAME OVER':'GAME OVER'}}</div>

        <!-- Duel winner -->
        <template v-if="gameMode==='tron'">
          <div class="go-tron-winner">{{tronWinner}}</div>
          <div class="go-who" style="margin-bottom:8px;">P1 🟢（WASD）vs 🟣 P2 {{is2PAI?'(🤖 AI)':'（⬆⬇⬅➡）'}}</div>
          <div class="go-sub">P1 剩 {{lives1}} 命 ｜ 得分 {{score1}} ｜ P2 剩 {{lives2}} 命 ｜ 得分 {{score2}}</div>
        </template>

        <!-- Stage mode result -->
        <template v-else-if="gameMode==='stage'">
          <div class="go-who">{{playerName||'玩家'}}</div>
          <div class="go-pts">{{stage}}<span class="go-unit"> / {{MAX_STAGE}} 關</span></div>
          <div class="go-sub">分數：{{score}} pts ｜ 獲得 🪙 {{Math.max(0,stage-1)*3}}</div>
        </template>

        <!-- Normal / Boss score -->
        <template v-else>
          <div class="go-who">{{isAI?'🤖 AI 模式':playerName}}</div>
          <div class="go-pts">{{score}}<span class="go-unit"> pts</span></div>
        </template>

        <div class="go-btns">
          <button class="btn-play" @click="restart">🔄 再玩一次</button>
          <button class="btn-ghost" @click="goModeDetail(gameMode)">📋 模式頁</button>
          <button class="btn-ghost" @click="goMenu">🏠 主選單</button>
        </div>
      </div>
    </div>

    <!-- ===== PLAYING ===== -->
    <div v-else class="game">
      <div class="game-wrap">
        <canvas ref="canvasRef" :width="canvasWidth" :height="canvasHeight" />

        <!-- Pause button -->
        <button class="btn-pause" @click="togglePause" :title="paused ? '繼續 (ESC)' : '暫停 (ESC)'">
          {{ paused ? '▶' : '⏸' }}
        </button>

        <!-- Pause overlay -->
        <transition name="pause-fade">
          <div v-if="paused" class="pause-overlay">
            <div class="pause-box">
              <div class="pause-icon">⏸</div>
              <div class="pause-title">PAUSED</div>
              <div class="pause-sub">遊戲已暫停</div>
              <div class="pause-btns">
                <button class="btn-play pause-resume" @click="resumeGame">▶ 繼續遊戲</button>
                <button class="btn-ghost" @click="goMenuFromPause">🏠 主選單</button>
              </div>
              <div class="pause-hint">按 ESC 或 P 也可繼續</div>
            </div>
          </div>
        </transition>

        <!-- Revive prompt -->
        <transition name="pause-fade">
          <div v-if="showRevivePrompt" class="pause-overlay">
            <div class="pause-box">
              <div class="pause-icon">❤️</div>
              <div class="pause-title">復活？</div>
              <div class="pause-sub">你有一個額外生命，要使用嗎？</div>
              <div class="pause-btns">
                <button class="btn-play pause-resume" @click="reviveYes">❤️ 使用（消耗）</button>
                <button class="btn-ghost" @click="reviveNo">✕ 不用了</button>
              </div>
            </div>
          </div>
        </transition>
      </div>
    </div>

  </div>
</template>

<style scoped>
* { box-sizing:border-box; }

.wrap {
  min-height:100vh;
  display:flex; justify-content:center; align-items:center;
  background:#020812;
  font-family:"Courier New",monospace;
  color:#fff;
}

.layout { display:flex; align-items:flex-start; gap:22px; }

/* sidebar */
.sidebar { width:185px; flex-shrink:0; display:flex; flex-direction:column; gap:11px; }

.lb-box {
  background:rgba(0,255,136,.04);
  border:1px solid rgba(0,255,136,.14);
  border-radius:12px; padding:12px 10px;
}
.lb-title { color:#00ffaa; font-size:12px; text-align:center; margin-bottom:10px; letter-spacing:2px; }
.lb-empty { color:rgba(255,255,255,.2); font-size:11px; text-align:center; padding:6px 0; }
.lb-row {
  display:flex; align-items:center; gap:5px;
  padding:6px; border-radius:7px; margin-bottom:4px;
  background:rgba(255,255,255,.04); font-size:11px;
}
.r1{background:rgba(255,213,74,.12);border:1px solid rgba(255,213,74,.28);}
.r2{background:rgba(192,192,192,.09);border:1px solid rgba(192,192,192,.2);}
.r3{background:rgba(205,127,50,.09);border:1px solid rgba(205,127,50,.2);}
.lb-medal{width:20px;text-align:center;font-size:13px;}
.lb-name{flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;color:#ccc;}
.lb-score{font-weight:bold;color:#00ffaa;font-size:12px;}

.hist-box {
  background:rgba(255,255,255,.03);
  border:1px solid rgba(255,255,255,.07);
  border-radius:12px; padding:10px 8px;
}
.hist-title{font-size:11px;color:rgba(255,255,255,.3);text-align:center;margin-bottom:7px;letter-spacing:1px;}
.hist-scroll{max-height:220px;overflow-y:auto;}
.hist-card{
  display:flex;align-items:center;gap:4px;
  padding:6px 7px;margin-bottom:4px;
  background:rgba(255,255,255,.06);border-radius:7px;
  cursor:pointer;font-size:11px;transition:.15s;
}
.hist-card:hover{background:rgba(0,255,136,.1);transform:scale(1.03);}
.hc-idx{color:rgba(255,255,255,.3);white-space:nowrap;}
.hc-who{flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;color:#ddd;}
.hc-score{color:#fbd74b;font-weight:bold;white-space:nowrap;}

/* center */
.center { text-align:center; min-width:260px; }
.main-center { min-width:360px; }

.logo { display:flex; align-items:center; justify-content:center; gap:10px; margin-bottom:2px; }
.logo-txt { font-size:26px; letter-spacing:5px; color:#00ffaa; text-shadow:0 0 20px #00ffaa88; }
.logo-snake { font-size:30px; }
.tagline { font-size:10px; letter-spacing:7px; color:rgba(0,255,170,.35); margin-bottom:24px; }

/* mode cards (main menu) */
.mode-cards { display:flex; flex-direction:column; gap:12px; align-items:center; }
.mode-card {
  display:flex; align-items:center; gap:14px;
  width:320px; padding:16px 20px;
  background:rgba(0,255,136,.04);
  border:1px solid rgba(0,255,136,.14);
  border-radius:14px; cursor:pointer; transition:.25s;
}
.mode-card:hover{background:rgba(0,255,136,.1);border-color:#00ffaa;transform:scale(1.03);box-shadow:0 0 24px rgba(0,255,136,.15);}
.mc-icon{font-size:32px;flex-shrink:0;}
.mc-title{font-size:15px;font-weight:bold;color:#fff;letter-spacing:1px;display:block;margin-bottom:2px;}
.mc-desc{font-size:10px;color:rgba(255,255,255,.35);display:block;}

/* color picker */
.color-picker{margin-top:18px;text-align:center;}
.cp-label{font-size:10px;color:rgba(255,255,255,.25);letter-spacing:2px;margin-bottom:8px;}
.cp-row{display:flex;justify-content:center;gap:8px;}
.cp-swatch{
  width:26px;height:26px;border-radius:50%;
  border:2px solid transparent;cursor:pointer;
  transition:.2s;display:inline-block;
}
.cp-swatch:hover{transform:scale(1.25);}
.cp-swatch.active{border-color:#fff;box-shadow:0 0 14px rgba(255,255,255,.35);}

/* back button */
.back-btn {
  background:transparent; border:1px solid rgba(255,255,255,.1);
  color:rgba(255,255,255,.3); font-size:12px; padding:6px 14px;
  border-radius:8px; cursor:pointer; transition:.15s;
  font-family:inherit; margin-bottom:16px;
}
.back-btn:hover{color:#fff;border-color:rgba(255,255,255,.25);}

/* mode hero (detail page) */
.mode-hero{margin-bottom:16px;text-align:center;}
.mh-icon{font-size:36px;display:block;margin-bottom:6px;}
.mh-title{font-size:16px;font-weight:bold;color:#fff;letter-spacing:2px;margin-bottom:6px;}
.mh-desc{font-size:10px;color:rgba(255,255,255,.4);line-height:1.7;max-width:280px;margin:0 auto;}

/* section divider */
.section-title{font-size:11px;color:rgba(255,255,255,.35);margin:10px 0 6px;letter-spacing:2px;}
.section-divider{font-size:10px;color:rgba(255,255,255,.2);letter-spacing:3px;margin:12px 0 8px;border-top:1px solid rgba(255,255,255,.06);padding-top:12px;}
.name-row{display:flex;gap:10px;justify-content:center;margin-bottom:10px;}
.online-status{text-align:center;margin-top:8px;}
.ol-msg{font-size:11px;color:#fbd74b;letter-spacing:1px;text-shadow:0 0 8px rgba(251,215,75,.3);}

/* ctrl hint */
.ctrl-hint{font-size:10px;color:rgba(255,255,255,.25);margin-top:6px;letter-spacing:1px;}

/* shop bar (main menu) */
.shop-bar{display:flex;align-items:center;justify-content:center;gap:10px;margin-top:14px;}
.shop-coins{font-size:14px;color:#fbd74b;letter-spacing:1px;text-shadow:0 0 8px rgba(251,215,75,.3);}
.btn-shop{
  background:rgba(251,215,75,.12);border:1px solid rgba(251,215,75,.3);
  color:#fbd74b;padding:7px 16px;font-size:12px;font-weight:bold;
  border-radius:8px;cursor:pointer;transition:.2s;font-family:inherit;letter-spacing:1px;
}
.btn-shop:hover{background:rgba(251,215,75,.22);transform:scale(1.06);}

/* shop page */
.shop-coins-display{font-size:22px;color:#fbd74b;text-shadow:0 0 12px rgba(251,215,75,.3);margin-bottom:16px;}
.shop-items{display:flex;flex-direction:column;gap:8px;align-items:center;}
.shop-item{
  display:flex;align-items:center;gap:10px;
  width:300px;padding:10px 14px;
  background:rgba(255,255,255,.04);border:1px solid rgba(255,255,255,.08);
  border-radius:10px;text-align:left;
}
.shop-item.owned{opacity:0.6;border-color:rgba(0,255,136,.2);}
.si-icon{font-size:24px;flex-shrink:0;}
.si-body{flex:1;}
.si-name{font-size:12px;font-weight:bold;color:#fff;margin-bottom:2px;}
.si-desc{font-size:9px;color:rgba(255,255,255,.4);}
.btn-buy{
  background:rgba(251,215,75,.12);border:1px solid rgba(251,215,75,.3);
  color:#fbd74b;padding:6px 10px;font-size:11px;font-weight:bold;
  border-radius:6px;cursor:pointer;transition:.15s;font-family:inherit;white-space:nowrap;
}
.btn-buy:hover{background:rgba(251,215,75,.25);}
.btn-buy.cant{opacity:0.4;cursor:not-allowed;background:transparent;}
.si-owned{font-size:10px;color:#00ffaa;white-space:nowrap;}

/* game over sub */
.go-sub{font-size:11px;color:rgba(255,255,255,.3);margin-bottom:16px;}

/* LB mode tag */
.lb-mode{font-size:9px;color:rgba(255,255,255,.2);display:block;margin-top:2px;}

/* Tron winner */
.go-tron-winner{font-size:28px;font-weight:bold;color:#fbd74b;text-shadow:0 0 24px #fbd74b66;margin:12px 0;}

/* power-up icons */
.powers-row { display:flex; justify-content:center; gap:14px; margin-bottom:10px; font-size:22px; }
.pp {
  transition:.2s; cursor:pointer;
  padding:5px; border-radius:8px;
  border:2px solid transparent;
  user-select:none;
}
.pp:hover { transform:scale(1.35) rotate(-10deg); }
.pp.active {
  background:rgba(255,255,255,.08);
  border-color:rgba(255,255,255,.25);
  transform:scale(1.2);
}

/* power-up tooltip */
.pu-tooltip {
  display:flex; align-items:center; gap:12px;
  background:rgba(5,10,24,.92);
  border:1px solid;
  border-radius:12px;
  padding:11px 14px;
  margin-bottom:14px;
  text-align:left;
  position:relative;
}
.pu-tt-icon { font-size:28px; flex-shrink:0; }
.pu-tt-body { flex:1; }
.pu-tt-name { font-size:11px; font-weight:bold; letter-spacing:2px; margin-bottom:4px; }
.pu-tt-desc { font-size:11px; color:rgba(255,255,255,.6); line-height:1.55; }
.pu-tt-close {
  position:absolute; top:7px; right:9px;
  background:transparent; border:none; color:rgba(255,255,255,.25);
  cursor:pointer; font-size:12px; padding:2px 4px; transition:.15s;
  font-family:inherit;
}
.pu-tt-close:hover { color:rgba(255,255,255,.7); }

.pu-fade-enter-active, .pu-fade-leave-active { transition:opacity .2s, transform .2s; }
.pu-fade-enter-from, .pu-fade-leave-to { opacity:0; transform:translateY(-6px); }

.name-wrap { display:flex; flex-direction:column; align-items:center; gap:5px; margin-bottom:16px; }
.name-lbl { font-size:10px; color:rgba(255,255,255,.3); letter-spacing:2px; }
.name-inp {
  background:rgba(255,255,255,.06);
  border:1px solid rgba(0,255,136,.22);
  border-radius:8px; color:#fff;
  padding:8px 14px; font-size:15px;
  text-align:center; width:195px; outline:none;
  font-family:inherit; transition:.2s;
}
.name-inp::placeholder{color:rgba(255,255,255,.18);}
.name-inp:focus{border-color:#00ffaa;box-shadow:0 0 12px rgba(0,255,136,.2);}
.name-inp.err{border-color:#f87171;}
.name-err{font-size:11px;color:#f87171;}

.btn-row { display:flex; gap:10px; justify-content:center; margin-bottom:20px; }

.btn-play {
  background:linear-gradient(135deg,#00cc88,#009960);
  color:#000; border:none; padding:11px 22px;
  font-size:14px; font-weight:bold; border-radius:10px;
  cursor:pointer; transition:.2s; font-family:inherit; letter-spacing:1px;
  box-shadow:0 0 18px rgba(0,255,136,.28);
}
.btn-play:hover{transform:scale(1.06);box-shadow:0 0 28px rgba(0,255,136,.5);}

.btn-ai {
  background:rgba(192,132,252,.14);
  color:#c084fc; border:1px solid rgba(192,132,252,.38);
  padding:11px 18px; font-size:14px; font-weight:bold;
  border-radius:10px; cursor:pointer; transition:.2s;
  font-family:inherit; letter-spacing:1px;
}
.btn-ai:hover{background:rgba(192,132,252,.26);transform:scale(1.06);box-shadow:0 0 18px rgba(192,132,252,.3);}

.btn-ghost {
  background:transparent; color:rgba(255,255,255,.38);
  border:1px solid rgba(255,255,255,.12);
  padding:11px 22px; font-size:13px; font-weight:bold;
  border-radius:10px; cursor:pointer; transition:.2s; font-family:inherit;
}
.btn-ghost:hover{color:#fff;border-color:rgba(255,255,255,.3);}

.tip-grid { display:grid; grid-template-columns:1fr 1fr; gap:5px 10px; margin-top:4px; }
.tip { display:flex; align-items:center; gap:5px; font-size:11px; }
.tip-txt { color:rgba(255,255,255,.3); }

/* game over */
.go-title {
  font-size:34px; font-weight:bold; letter-spacing:5px;
  color:#ff4060; text-shadow:0 0 24px #ff004088;
  margin-bottom:8px;
}
.go-who{font-size:14px;color:rgba(255,255,255,.4);margin-bottom:6px;letter-spacing:1px;}
.go-pts{font-size:62px;font-weight:bold;color:#fbd74b;text-shadow:0 0 30px #ffa00088;line-height:1;margin-bottom:24px;}
.go-unit{font-size:22px;color:rgba(255,255,255,.28);}
.go-btns{display:flex;flex-direction:column;gap:10px;align-items:center;}
.go-btns button{width:200px;}

/* canvas / game */
.game{display:flex;justify-content:center;align-items:center;}

.game-wrap {
  position:relative;
  display:inline-block;
}

canvas{
  display:block;
  border-radius:12px;
  box-shadow:0 0 40px rgba(0,255,136,.12),0 0 80px rgba(0,0,0,.6);
}

/* pause button */
.btn-pause {
  position:absolute;
  top:10px; right:10px;
  width:34px; height:34px;
  background:rgba(0,0,0,.55);
  border:1px solid rgba(255,255,255,.18);
  border-radius:8px;
  color:rgba(255,255,255,.7);
  font-size:15px;
  cursor:pointer;
  display:flex; align-items:center; justify-content:center;
  transition:.15s;
  z-index:10;
}
.btn-pause:hover {
  background:rgba(255,255,255,.12);
  color:#fff;
  border-color:rgba(255,255,255,.35);
}

/* pause overlay */
.pause-overlay {
  position:absolute;
  inset:0;
  border-radius:12px;
  background:rgba(2,8,18,.82);
  backdrop-filter:blur(4px);
  display:flex; align-items:center; justify-content:center;
  z-index:20;
}

.pause-box {
  text-align:center;
  padding:32px 36px;
  background:rgba(0,255,136,.04);
  border:1px solid rgba(0,255,136,.18);
  border-radius:18px;
  box-shadow:0 0 40px rgba(0,255,136,.1);
}

.pause-icon {
  font-size:36px;
  margin-bottom:8px;
  filter:drop-shadow(0 0 12px rgba(0,255,170,.6));
}

.pause-title {
  font-size:28px;
  font-weight:bold;
  letter-spacing:6px;
  color:#00ffaa;
  text-shadow:0 0 20px rgba(0,255,170,.5);
  margin-bottom:4px;
}

.pause-sub {
  font-size:11px;
  color:rgba(255,255,255,.3);
  letter-spacing:2px;
  margin-bottom:24px;
}

.pause-btns {
  display:flex;
  flex-direction:column;
  gap:10px;
  align-items:center;
}

.pause-resume {
  width:200px;
}

.pause-btns .btn-ghost {
  width:200px;
}

.pause-hint {
  margin-top:16px;
  font-size:10px;
  color:rgba(255,255,255,.18);
  letter-spacing:1px;
}

/* pause transition */
.pause-fade-enter-active, .pause-fade-leave-active {
  transition:opacity .2s, transform .2s;
}
.pause-fade-enter-from, .pause-fade-leave-to {
  opacity:0;
  transform:scale(.95);
}
</style>