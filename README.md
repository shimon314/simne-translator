<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Simne Translator</title>

<style>
:root{
  --bg:#0f172a;
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
}
main{
  padding:12px;
  display:grid;
  grid-template-columns:2fr 1fr;
  gap:12px;
}
.card{
  background:var(--card);
  border:1px solid var(--border);
  border-radius:12px;
  padding:12px;
}
textarea,input{
  width:100%;
  background:transparent;
  border:none;
  color:var(--text);
  font-size:16px;
  outline:none;
}
.output{white-space:pre-wrap}
.unknown{color:var(--err);text-decoration:underline}
.label{font-size:12px;color:var(--sub);margin-bottom:6px}
@media(max-width:900px){
  main{grid-template-columns:1fr}
}
</style>
</head>

<body>

<header>
  <b>Simne Translator</b>
  <div id="status" class="label">辞書ロード中…</div>
</header>

<main>

<div class="card">
  <div class="label">日本語</div>
  <textarea id="input" placeholder="私はあなたの本を見る"></textarea>
  <hr>
  <div class="label">シムネ語</div>
  <div id="output" class="output"></div>
</div>

<div class="card">
  <div class="label">辞書検索</div>
  <input id="search" placeholder="日本語 or シムネ語">
  <div id="dictResult" class="output"></div>
</div>

</main>

<script>
/* ====== 設定 ====== */
const DICT_FILE="シムネ語 詩日辞書.json";
const COPULA={first:"seg",second:"sem",third:"set"};
const COPULA_TRIG=["だ","です","である"];
const FIRST=["私","僕","俺"];
const SECOND=["あなた","君"];

/* ====== 格 ====== */
const CASE_TABLE={
  ce:{nom:"ce",gen:"ces",acc:"cem"},
  di:{nom:"di",gen:"dit",acc:"dim"},
  ra:{nom:"ra",gen:"rai",acc:"rat"},
  da:{nom:"da",gen:"dai",acc:"dat"},
  ya:{nom:"ya",gen:"yai",acc:"yat"},
  o:{nom:"o",gen:"ok",acc:"e"}
};

/* ====== 状態 ====== */
let dict={},ready=false;

/* ====== 初期化 ====== */
loadDict();
input.oninput=translate;
search.oninput=searchDict;

/* ====== 辞書 ====== */
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
    status.textContent=`辞書ロード完了：${Object.keys(dict).length}語`;
  }catch{
    status.textContent="辞書ロード失敗";
  }
}

/* ====== 辞書検索 ====== */
function searchDict(){
  const q=search.value.trim();
  if(!q){dictResult.textContent="";return;}
  const res=[];
  for(const [ja,si] of Object.entries(dict)){
    if(ja.includes(q)||si.includes(q)){
      res.push(`${ja} → ${si}`);
    }
  }
  dictResult.textContent=res.join("\n")||"見つかりません";
}

/* ====== 分割 ====== */
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
    if(!hit){out.push(t[i]);i++;}
  }
  return out;
}

/* ====== 時制自動判定 ====== */
function detectTense(t){
  if(/した|だった/.test(t)) return "past";
  if(/だろう|つもり/.test(t)) return "future";
  return "present";
}

/* ====== 格 ====== */
function applyCase(w,k){
  if(CASE_TABLE[w]) return CASE_TABLE[w][k];
  if(k==="gen") return w+"d";
  return w;
}

/* ====== 翻訳 ====== */
function translate(){
  if(!ready)return;
  const t=input.value.trim();
  if(!t){output.textContent="";return;}

  const tense=detectTense(t);
  const ws=t.includes(" ")?t.split(/\s+/):tokenize(t);

  let S=null,V=null,O=null,possessor=null,cop=false;

  for(let i=0;i<ws.length;i++){
    const w=ws[i];
    if(["は","が"].includes(w)) continue;
    if(w==="の"){ possessor=ws[i-1]; continue; }
    if(w==="を"){ O=ws[i-1]; continue; }
    if(COPULA_TRIG.includes(w)){ cop=true; continue; }
    if(!S) S=w;
    else if(!V) V=w;
  }

  const person=FIRST.includes(S)?"first":SECOND.includes(S)?"second":"third";
  const out=[];

  // 主語
  out.push(applyCase(dict[S]||u(S),"nom"));

  // コピュラ
  if(cop){
    let c=COPULA[person];
    if(tense==="past")c+="s";
    if(tense==="future")c+="it";
    out.push(c);
  }

  // 目的語（所有格を内部処理）
  if(O){
    let obj=dict[O]||u(O);
    if(possessor){
      let p=dict[possessor]||u(possessor);
      out.push(applyCase(p,"gen"), obj);
    }else{
      out.push(applyCase(obj,"acc"));
    }
  }

  output.innerHTML=out.join(" ");
}

function u(w){return `<span class="unknown">${w}</span>`;}
</script>

</body>
</html>
