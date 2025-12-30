<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>シムネ語 翻訳エンジン</title>

<style>
:root {
  --bg:#0e1117;
  --panel:#161b22;
  --border:#30363d;
  --text:#e6edf3;
  --ok:#3fb950;
  --err:#f85149;
  --accent:#238636;
}
body {
  background:var(--bg);
  color:var(--text);
  font-family:-apple-system,BlinkMacSystemFont,sans-serif;
  padding:14px;
}
h1 {
  font-size:22px;
  margin-bottom:6px;
}
#status {
  font-size:13px;
  margin-bottom:10px;
}
.ok { color:var(--ok); }
.err { color:var(--err); }

.panel {
  background:var(--panel);
  border:1px solid var(--border);
  border-radius:8px;
  padding:12px;
  margin-bottom:12px;
}

textarea {
  width:100%;
  background:transparent;
  color:var(--text);
  border:none;
  resize:none;
  font-size:16px;
  min-height:72px;
  outline:none;
}

.toggle {
  display:flex;
  gap:6px;
  margin-top:10px;
}
.toggle button {
  flex:1;
  padding:8px;
  border-radius:6px;
  border:1px solid var(--border);
  background:#0d1117;
  color:var(--text);
}
.toggle button.active {
  background:var(--accent);
}

#output span.unknown {
  color:var(--err);
  text-decoration:underline;
}
</style>
</head>

<body>

<h1>シムネ語 翻訳エンジン</h1>
<div id="status">辞書ロード中…</div>

<div class="panel">
<textarea id="input" placeholder="私は学生だ / 私は世界を見る"></textarea>

<div class="toggle">
  <button data-tense="present" class="active">現在</button>
  <button data-tense="past">過去</button>
  <button data-tense="future">未来</button>
</div>
</div>

<div class="panel">
<b>翻訳結果</b>
<div id="output"></div>
</div>

<script>
/* ===== 設定 ===== */
const DICT_FILE = "シムネ語 詩日辞書.json";
const COPULA = { first:"seg", second:"sem", third:"set" };
const COPULA_TRIGGERS = ["だ","です","である"];
const FIRST = ["私","僕","俺"];
const SECOND = ["あなた","君"];

/* ===== 状態 ===== */
let jaToSimne = {};
let dictReady = false;
let currentTense = "present";

/* ===== 初期化 ===== */
loadDictionary();
bindUI();

/* ===== 辞書ロード ===== */
async function loadDictionary() {
  try {
    const res = await fetch(DICT_FILE,{cache:"no-store"});
    const data = await res.json();

    data.words.forEach(w=>{
      const sim = w.entry?.form;
      if (!sim) return;
      (w.translations||[]).forEach(tr=>{
        if (!tr.rawForms) return;
        tr.rawForms
          .replace(/（.*?）|\(.*?\)/g,"")
          .split(/[、,]/)
          .forEach(j=>{
            const k=j.trim();
            if(k) jaToSimne[k]=sim;
          });
      });
    });

    dictReady = true;
    setStatus(`辞書ロード完了：${Object.keys(jaToSimne).length}語`,true);
  } catch {
    setStatus("辞書ロード失敗",false);
  }
}

/* ===== UI ===== */
function bindUI() {
  document.getElementById("input")
    .addEventListener("input",translate);

  document.querySelectorAll(".toggle button").forEach(b=>{
    b.onclick=()=>{
      document.querySelectorAll(".toggle button")
        .forEach(x=>x.classList.remove("active"));
      b.classList.add("active");
      currentTense=b.dataset.tense;
      translate();
    };
  });
}

function setStatus(text,ok){
  document.getElementById("status").innerHTML =
    `<span class="${ok?"ok":"err"}">${text}</span>`;
}

/* ===== 日本語分割 ===== */
function tokenize(text){
  const keys=Object.keys(jaToSimne).sort((a,b)=>b.length-a.length);
  const out=[];
  let i=0;
  while(i<text.length){
    let hit=false;
    for(const k of keys){
      if(text.startsWith(k,i)){
        out.push(k); i+=k.length; hit=true; break;
      }
    }
    if(!hit){ out.push(text[i]); i++; }
  }
  return out;
}

/* ===== 翻訳 ===== */
function translate(){
  if(!dictReady) return;

  const text=document.getElementById("input").value.trim();
  if(!text){
    document.getElementById("output").innerHTML="";
    return;
  }

  const words=text.includes(" ")
    ? text.split(/\s+/)
    : tokenize(text);

  let subject=null, comps=[], cop=false;

  words.forEach(w=>{
    if(["は","が"].includes(w)) return;
    if(COPULA_TRIGGERS.includes(w)){ cop=true; return; }
    if(!subject) subject=w;
    else comps.push(w);
  });

  const person = FIRST.includes(subject)
    ? "first"
    : SECOND.includes(subject)
      ? "second"
      : "third";

  const out=[];
  out.push(jaToSimne[subject]||u(subject));

  if(cop){
    let c=COPULA[person];
    if(currentTense==="past") c+="s";
    if(currentTense==="future") c+="it";
    out.push(c);
  }

  comps.forEach(w=>{
    out.push(jaToSimne[w]||u(w));
  });

  document.getElementById("output").innerHTML=out.join(" ");
}

function u(w){ return `<span class="unknown">${w}</span>`; }
</script>

</body>
</html>
