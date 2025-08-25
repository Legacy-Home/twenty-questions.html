<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>20 Questions — Classroom Edition</title>
<style>
  :root{
    --bg:#0f172a; --card:#111827; --muted:#94a3b8;
    --text:#e5e7eb; --accent:#22c55e; --accent2:#60a5fa; --danger:#f87171; --warn:#f59e0b;
  }
  *{box-sizing:border-box}
  html,body{height:100%}
  body{
    margin:0; font-family:system-ui,-apple-system,Segoe UI,Roboto,Inter,Arial,sans-serif;
    background:radial-gradient(1200px 800px at 20% -10%, #1f2937 0%, var(--bg) 50%),
               radial-gradient(1400px 900px at 120% 10%, #0b1220 0%, var(--bg) 60%);
    color:var(--text); display:flex; align-items:center; justify-content:center; padding:24px;
  }
  .app{
    width:min(960px,100%); background:linear-gradient(180deg,#0b1220 0%,#0b1220cc 35%,#0b122080 100%),var(--card);
    border:1px solid #1f2937; border-radius:16px; padding:20px; box-shadow:0 10px 30px #0008;
  }
  h1{margin:0 0 6px; font-size:28px}
  .subtitle{color:var(--muted); margin-bottom:18px}
  .row{display:flex; gap:12px; flex-wrap:wrap}
  .card{background:#0b1220; border:1px solid #1f2937; border-radius:12px; padding:14px}
  .controls{display:flex; gap:10px; flex-wrap:wrap}
  button{
    appearance:none; border:none; border-radius:10px; padding:12px 16px; font-weight:600; cursor:pointer;
    color:#091016; background:var(--accent); box-shadow:0 2px 0 #0e3f27 inset;
  }
  button.secondary{background:#111827; color:var(--text); border:1px solid #1f2937; box-shadow:none}
  button.warn{background:var(--warn); color:#1a1206}
  button.danger{background:var(--danger); color:#260b0b}
  button.gray{background:#374151; color:#e5e7eb}
  button:disabled{opacity:.5; cursor:not-allowed}
  .meter{display:flex; align-items:center; gap:8px}
  .bar{flex:1; height:8px; background:#111827; border:1px solid #1f2937; border-radius:6px; overflow:hidden}
  .fill{height:100%; background:linear-gradient(90deg,var(--accent2),#22d3ee); width:0%}
  .pill{display:inline-block; padding:6px 10px; border-radius:999px; background:#0b1220; border:1px solid #1f2937; color:var(--muted); margin:4px 6px 0 0; font-size:13px}
  .question{font-size:20px; margin:12px 0 10px}
  .summary{display:flex; gap:8px; flex-wrap:wrap; margin-top:6px}
  details{border:1px dashed #374151; border-radius:10px; padding:10px; margin-top:10px}
  .foot{margin-top:16px; color:var(--muted); font-size:13px}
  .big{font-size:24px}
  .muted{color:var(--muted)}
  .center{display:flex; align-items:center; justify-content:center}
</style>
</head>
<body>
  <div class="app" role="application" aria-label="20 Questions Classroom Game">
    <h1>20 Questions — Classroom Edition 🕵️‍♀️</h1>
    <div class="subtitle">Students think of something. You answer only <b>Yes</b> or <b>No</b>. We get <b>20</b> questions to narrow it down!</div>

    <div class="row">
      <div class="card" style="flex:2; min-width:260px">
        <div class="meter">
          <div><b id="count">0</b>/20</div>
          <div class="bar" aria-hidden="true"><div class="fill" id="fill"></div></div>
          <div id="remain">20 left</div>
        </div>

        <div id="question" class="question big">When you’re ready, click “Start” and I’ll ask broad → specific.</div>
        <div class="controls">
          <button id="yesBtn" disabled>Yes</button>
          <button id="noBtn"  disabled class="secondary">No</button>
          <button id="skipBtn" disabled class="gray" title="Skip if unclear">Skip</button>
          <button id="backBtn" disabled class="secondary" title="Undo last answer">Undo</button>
          <button id="resetBtn" class="danger" title="Start over">Reset</button>
          <button id="startBtn" class="warn">Start</button>
        </div>

        <details>
          <summary>Teacher tip</summary>
          <ul style="margin:8px 0 0 18px; line-height:1.5">
            <li>Keep answers to “Yes” or “No”. If unclear, click <b>Skip</b>.</li>
            <li>Pause after 5, 10, 15 questions to ask: “What do we know so far?”</li>
            <li>Wrap‑up: We needed many questions. Contrast that with God’s omniscience—He doesn’t need questions to know our thoughts (Psalm 139).</li>
          </ul>
        </details>
      </div>

      <div class="card" style="flex:1; min-width:240px">
        <div><b>What we’ve learned</b></div>
        <div class="summary" id="facts"></div>
        <details>
          <summary>Possible guesses (may be wrong!)</summary>
          <div id="guesses" class="summary"></div>
        </details>
      </div>
    </div>

    <div class="foot">
      This tool is limited and sometimes silly. That’s the point: humans (and AI!) are guessers—God is not.
    </div>
  </div>

<script>
/* --- Minimal knowledge model + question flow --- */
const MAX_Q = 20;

const state = {
  qIndex: 0,
  asked: [],
  answers: {}, // key -> true/false
  history: [], // stack of {key, answer}
  candidates: [] // narrowed items
};

// Simple ontology for suggestions (tiny on purpose)
const ITEMS = [
  // persons
  {name:"Teacher", tags:["person","human","living","not_animal","indoors","voice","common"]},
  {name:"Doctor", tags:["person","human","living","not_animal","indoors","tools","common"]},
  {name:"Athlete", tags:["person","human","living","not_animal","outdoors","common"]},
  // animals
  {name:"Dog", tags:["animal","living","pet","outdoors","indoors","common","sound"]},
  {name:"Cat", tags:["animal","living","pet","indoors","common","sound"]},
  {name:"Elephant", tags:["animal","living","wild","large","outdoors"]},
  // places
  {name:"Classroom", tags:["place","indoors","school","common"]},
  {name:"Library", tags:["place","indoors","quiet","common"]},
  {name:"Beach", tags:["place","outdoors","water","sand"]},
  {name:"Mountain", tags:["place","outdoors","large"]},
  {name:"The Moon", tags:["place","outdoors","space"]},
  // things (home objects)
  {name:"Bed", tags:["object","indoors","home","bedroom","large","furniture"]},
  {name:"Chair", tags:["object","indoors","home","furniture"]},
  {name:"Table", tags:["object","indoors","home","furniture"]},
  {name:"Sofa", tags:["object","indoors","home","furniture","large"]},
  {name:"Lamp", tags:["object","indoors","home","electronic","light"]},
  {name:"Refrigerator", tags:["object","indoors","home","kitchen","electronic","large"]},
  {name:"Microwave", tags:["object","indoors","home","kitchen","electronic"]},
  {name:"Toothbrush", tags:["object","indoors","home","bathroom","small"]},
  {name:"Book", tags:["object","indoors","paper","small","quiet"]},
  {name:"TV", tags:["object","indoors","home","electronic","screen","large"]},
  {name:"Computer", tags:["object","indoors","electronic","screen"]},
  {name:"Phone", tags:["object","indoors","electronic","screen","small"]},
  {name:"Door", tags:["object","indoors","home","wood"]},
  {name:"Window", tags:["object","indoors","home","glass"]},
  {name:"Rug", tags:["object","indoors","home","fabric"]},
  {name:"Dresser", tags:["object","indoors","home","bedroom","furniture"]},
  {name:"Mirror", tags:["object","indoors","home","glass"]},
  {name:"Pillow", tags:["object","indoors","home","bedroom","fabric","small"]},
];

const QUESTIONS = [
  // phase 1: broad
  {key:"living", text:"Is it living?"},
  {key:"person", text:"Is it a person?"},
  {key:"animal", text:"Is it an animal?"},
  {key:"place", text:"Is it a place?"},
  {key:"object", text:"Is it a non-living object?"},
  // phase 2: size/location
  {key:"indoors", text:"Is it usually found indoors?"},
  {key:"outdoors", text:"Is it usually found outdoors?"},
  {key:"home", text:"Would you find it in a home?"},
  {key:"bedroom", text:"Would you find it in a bedroom?"},
  {key:"kitchen", text:"Would you find it in a kitchen?"},
  {key:"school", text:"Is it associated with school?"},
  // phase 3: qualities/usage
  {key:"electronic", text:"Does it use electricity?"},
  {key:"screen", text:"Does it have a screen?"},
  {key:"furniture", text:"Is it furniture?"},
  {key:"large", text:"Is it larger than a backpack?"},
  {key:"small", text:"Is it small enough to hold in one hand?"},
  {key:"wood", text:"Is it made mostly of wood?"},
  {key:"glass", text:"Does it have glass as a main part?"},
  {key:"fabric", text:"Is it made mostly of fabric?"},
  {key:"sound", text:"Does it make a distinct sound?"},
];

const PREREQS = {
  person: {living:true},
  animal: {living:true, person:false},
  object: {living:false},
  bedroom: {home:true}, kitchen:{home:true},
  screen: {electronic:true},
  furniture: {object:true},
};

function prereqsMet(key){
  const req = PREREQS[key];
  if(!req) return true;
  for(const [k,val] of Object.entries(req)){
    if(state.answers[k] === undefined) return false;
    if(state.answers[k] !== val) return false;
  }
  return true;
}

// choose next question: unasked, prereqs met, and not logically redundant
function nextQuestion(){
  for(const q of QUESTIONS){
    if(state.asked.includes(q.key)) continue;
    if(!prereqsMet(q.key)) continue;
    return q;
  }
  return null;
}

function updateCandidates(){
  const yes = Object.entries(state.answers).filter(([k,v])=>v).map(([k])=>k);
  const no  = Object.entries(state.answers).filter(([k,v])=>!v).map(([k])=>k);
  state.candidates = ITEMS.filter(item=>{
    for(const k of yes){ if(!item.tags.includes(k)) return false; }
    for(const k of no){ if(item.tags.includes(k)) return false; }
    return true;
  });
}

function factPills(){
  const facts = [];
  for(const [k,v] of Object.entries(state.answers)){
    const txt = keyLabel(k);
    if(v===true) facts.push(`Yes: ${txt}`);
    else if(v===false) facts.push(`No: ${txt}`);
  }
  return facts;
}

function keyLabel(k){
  const map = {
    living:"living", person:"person", animal:"animal", place:"place", object:"object",
    indoors:"indoors", outdoors:"outdoors", home:"in a home", bedroom:"in a bedroom",
    kitchen:"in a kitchen", school:"at school", electronic:"uses electricity", screen:"has a screen",
    furniture:"furniture", large:"larger than a backpack", small:"handheld",
    wood:"wood", glass:"glass", fabric:"fabric", sound:"makes a sound"
  };
  return map[k] || k;
}

/* --- UI wiring --- */
const $ = sel => document.querySelector(sel);
const countEl = $("#count"), remainEl=$("#remain"), fillEl=$("#fill");
const qEl = $("#question"), factsEl=$("#facts"), guessesEl=$("#guesses");
const yesBtn = $("#yesBtn"), noBtn = $("#noBtn"), skipBtn=$("#skipBtn"), backBtn=$("#backBtn");
const startBtn = $("#startBtn"), resetBtn=$("#resetBtn");

function render(){
  countEl.textContent = state.qIndex;
  remainEl.textContent = (MAX_Q - state.qIndex) + " left";
  fillEl.style.width = (state.qIndex / MAX_Q * 100) + "%";

  // facts
  factsEl.innerHTML = "";
  for(const text of factPills()){
    const span = document.createElement("span"); span.className="pill"; span.textContent = text; factsEl.appendChild(span);
  }

  // guesses
  updateCandidates();
  guessesEl.innerHTML = "";
  const top = state.candidates.slice(0,6);
  if(top.length){
    for(const c of top){
      const span = document.createElement("span"); span.className="pill"; span.textContent = c.name; guessesEl.appendChild(span);
    }
  } else {
    const span = document.createElement("span"); span.className="pill"; span.textContent = "No strong guesses yet — keep asking!";
    guessesEl.appendChild(span);
  }

  // question
  const nq = nextQuestion();
  if(state.qIndex >= MAX_Q || !nq){
    yesBtn.disabled = noBtn.disabled = skipBtn.disabled = true;
    backBtn.disabled = state.history.length===0;
    qEl.innerHTML = (state.qIndex >= MAX_Q)
      ? "That’s 20! Reveal your item and compare with our facts. 🎉<br><span class='muted'>Humans & AI ask questions. God already knows.</span>"
      : "No more helpful questions. Reveal your item and compare with our facts. 🎉";
  } else {
    qEl.textContent = nq.text;
    yesBtn.disabled = noBtn.disabled = skipBtn.disabled = false;
    backBtn.disabled = state.history.length===0;
  }
}

function answer(val){
  const nq = nextQuestion(); if(!nq) return;
  state.answers[nq.key] = val;
  state.asked.push(nq.key);
  state.history.push({key:nq.key, val});
  state.qIndex++;
  render();
}

function skip(){
  const nq = nextQuestion(); if(!nq) return;
  state.asked.push(nq.key);
  state.history.push({key:nq.key, val:null});
  state.qIndex++;
  render();
}

function back(){
  const last = state.history.pop();
  if(!last) return;
  const i = state.asked.lastIndexOf(last.key);
  if(i>=0) state.asked.splice(i,1);
  if(last.val !== null) delete state.answers[last.key];
  state.qIndex = Math.max(0, state.qIndex - 1);
  render();
}

function reset(){
  state.qIndex=0; state.asked=[]; state.answers={}; state.history=[]; state.candidates=[];
  render();
}

startBtn.addEventListener("click", render);
resetBtn.addEventListener("click", reset);
yesBtn.addEventListener("click", ()=>answer(true));
noBtn.addEventListener("click", ()=>answer(false));
skipBtn.addEventListener("click", skip);
backBtn.addEventListener("click", back);

// initial
render();
</script>
</body>
</html>
