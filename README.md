<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Simne Translator</title>

<style>
:root{
  --bg:#0f172a;
  --panel:#020617;
  --card:#020617;
  --border:#1e293b;
  --text:#e5e7eb;
  --sub:#94a3b8;
  --accent:#2563eb;
  --err:#ef4444;
}
*{box-sizing:border-box}
body{
  margin:0;
  background:var(--bg);
  color:var(--text);
  font-family:-apple-system,BlinkMacSystemFont,sans-serif;
}
header{
  padding:12px 16px;
  border-bottom:1px solid var(--border);
  font-weight:600;
}
#status{
  font-size:12px;
  color:var(--sub);
}
main{
  display:flex;
  flex-direction:column;
  gap:12px;
  padding:12px;
}
.translator{
  display:grid;
  grid-template-columns:1fr 1fr;
  gap:12px;
}
.card{
  background:var(--card);
  border:1px solid var(--border);
  border-radius:12px;
  padding:12px;
  display:flex;
  flex-direction:column;
}
.label{
  font-size:12px;
  color:var(--sub);
  margin-bottom:6px;
}
textarea{
  flex:1;
  background:transparent;
  border:none;
  color:var(--text);
  font-size:16px;
  resize:none;
  outline:none;
}
.output{
  font-size:16px;
  white-space:pre-wrap;
}
.unknown{
  color:var(--err);
  text-decoration:underline;
}
.controls{
  display:flex;
  gap:6px;
  margin-top:8px;
}
.controls button{
  flex:1;
  background:#020617;
  border:1px solid var(--border);
  color:var(--text);
  padding:6px;
  border-radius:8px;
}
.controls button.active{
  background:var(--accent);
}
@media(max-width:700px){
  .translator{grid-template-columns:1fr}
}
</style>
</head>

<body>

<header>
  Simne Translator
  <div id="status">Loading dictionary…</div>
</header>

<main>
  <div class="translator">

    <div class="card">
      <div class="label">日本語</div>
      <textarea id="input" placeholder="私はあなたの世界を見る"></textarea>
      <div class="controls">
        <button data-tense="present" class="active">現在</button>
        <button data-tense="past">過去</button>
        <button data-tense="future">未来</button>
      </div>
    </div>

    <div class="card">
      <div class="label">シムネ語</div>
      <div id="output" class="output"></div>
    </div>

  </div>
</main>

<script>
/* ========= 設定 ========= */
const DICT_FILE="シムネ語 詩日辞書.json";
const COPULA={first:"seg",second:"sem",third:"set"};
const COPULA_TRIG=["だ","です","である"];
const FIRST=["私","僕","俺"];
const SECOND=["あなた","君"];

/* ========= 格変化 ========= */
const CASE_TABLE={
  ce:{nom:"ce",gen:"ces",acc:"cem",poss:"cest"},
  di:{nom:"di",gen:"dit",acc:"dim",poss:"din"},
  ra:{nom:"ra",gen:"rai",acc:"rat",poss:"raks"},
  da:{nom:"da",gen:"dai",acc:"dat",poss:"daks"},
  ya:{nom:"ya",gen:"yai",acc:"yat",poss:"yake"},
  o:{nom:"o",gen:"ok",acc:"e",poss:"om"},
  ci:{nom:"ci",gen:"cis",acc:"cim",poss:"cit"},
  dis:{nom:"dis",gen:"dik",acc:"did",poss:"dip"},
  ro:{nom:"ro",gen:"re",acc:"ri",poss:"rori"},
  das:{nom:"das",gen:"do",acc:"dez",poss:"dao"},
  yo:{nom:"yo",gen:"yon",acc:"yom",poss:"yog"},
  oz:{nom:"oz",gen:"og",acc:"be",poss:"oks"}
};

/* ========= 状態 ========= */
let dict={},ready=false,tense="present";

/* ========= 初期化 ========= */
loadDict();
bindUI();

/* ========= 辞書 ========= */
async function loadDict(){
  try{
    const r=await fetch(DICT_FILE,{cache:"no-store"});
    const j=await r.json();
    j.words.forEach(w=>{
      const sim=w.entry?.form;
      if(!sim)return;
      (w.translations||[]).forEach(t=>{
        if(!t.rawForms)return;
        t.rawForms.replace(/（.*?）|\(.*?\)/g,"")
        .split(/[、,]/)
        .forEach(x=>{
          x=x.trim();
          if(x)dict[x]=sim;
        });
      });
    });
    ready=true;
    status(`辞書ロード完了：${Object.keys(dict).length}語`);
  }catch{
    status("辞書ロード失敗");
  }
}
function status(t){
  document.getElementById("status").textContent=t;
}

/* ========= UI ========= */
function bindUI(){
  input.oninput=translate;
  document.querySelectorAll(".controls button").forEach(b=>{
    b.onclick=()=>{
      document.querySelectorAll(".controls button")
        .forEach(x=>x.classList.remove("active"));
      b.classList.add("active");
      tense=b.dataset.tense;
      translate();
    };
  });
}

/* ========= 分割 ========= */
function tokenize(t){
  const keys=Object.keys(dict).sort((a,b)=>b.length-a.length);
  const out=[]; let i=0;
  while(i<t.length){
    let hit=false;
    for(const k of keys){
      if(t.startsWith(k,i)){
        out.push(k); i+=k.length; hit=true; break;
      }
    }
    if(!hit){ out.push(t[i]); i++; }
  }
  return out;
}

/* ========= 格 ========= */
function applyCase(w,k){
  if(CASE_TABLE[w]) return CASE_TABLE[w][k];
  if(k==="gen") return w+"d";
  if(k==="poss") return w+"s";
  return w;
}

/* ========= 翻訳 ========= */
function translate(){
  if(!ready){output.textContent="";return;}
  const t=input.value.trim();
  if(!t){output.textContent="";return;}

  const ws=t.includes(" ")?t.split(/\s+/):tokenize(t);
  let S=null,O=[],cop=false;

  ws.forEach(w=>{
    if(["は","が"].includes(w))return;
    if(w==="を"){return;}
    if(w==="の"){O.case="gen";return;}
    if(COPULA_TRIG.includes(w)){cop=true;return;}
    if(!S)S=w; else O.push(w);
  });

  const person=FIRST.includes(S)?"first":SECOND.includes(S)?"second":"third";
  let out=[];

  let s=dict[S]||u(S);
  out.push(applyCase(s,"nom"));

  if(cop){
    let c=COPULA[person];
    if(tense==="past")c+="s";
    if(tense==="future")c+="it";
    out.push(c);
  }

  O.forEach(w=>{
    let x=dict[w]||u(w);
    out.push(applyCase(x,"acc"));
  });

  output.innerHTML=out.join(" ");
}
function u(w){return `<span class="unknown">${w}</span>`;}
</script>

</body>
</html>
