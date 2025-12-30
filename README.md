<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Simne Translator (Stable Full)</title>

<style>
body{
  margin:0;
  background:#0f172a;
  color:#e5e7eb;
  font-family:-apple-system,BlinkMacSystemFont,sans-serif;
}
header{
  padding:12px 16px;
  border-bottom:1px solid #1e293b;
}
main{
  padding:12px;
  max-width:900px;
  margin:auto;
}
.card{
  background:#020617;
  border:1px solid #1e293b;
  border-radius:12px;
  padding:12px;
  margin-bottom:12px;
}
textarea{
  width:100%;
  min-height:80px;
  background:transparent;
  border:none;
  color:#e5e7eb;
  font-size:16px;
  outline:none;
}
.output{
  font-size:18px;
  white-space:pre-wrap;
}
.note{
  font-size:13px;
  color:#94a3b8;
}
.unknown{
  color:#f87171; /* 赤っぽい */
  font-weight:600;
}
.tag{
  color:#60a5fa;
  font-weight:600;
}
</style>
</head>

<body>

<header>
  <b>Simne Translator</b>
  <div class="note" id="status">辞書ロード中…</div>
</header>

<main>

<div class="card">
  <textarea id="inputText" placeholder="私は学生だ / 私は世界を見る" autocomplete="off" autocapitalize="off"></textarea>
  <div class="note">（入力中に即時翻訳されます。IME確定中は処理を待ちます）</div>
</div>

<div class="card">
  <div id="outputText" class="output"></div>
</div>

</main>

<script>
const inputEl  = document.getElementById("inputText");
const outputEl = document.getElementById("outputText");
const statusEl = document.getElementById("status");

/* コピュラ・人称 */
const PRONOUN_PERSON = {
  "私":"first","僕":"first","俺":"first",
  "あなた":"second","君":"second"
};
const COPULA = { first:"seg", second:"sem", third:"set" };

/* 辞書 */
let dict = {};
let ready = false;

/* IME / デバウンス制御 */
let composing = false;
let debounceTimer = null;
const DEBOUNCE_MS = 60;

/* 初期化 */
loadDict();
inputEl.addEventListener("input", ()=>{ if(!composing) debounceTranslate(); });
inputEl.addEventListener("compositionstart", ()=>{ composing = true; });
inputEl.addEventListener("compositionend", ()=>{ composing = false; debounceTranslate(); });

function debounceTranslate(){
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(translate, DEBOUNCE_MS);
}

/* エスケープ（安全に innerHTML にするため） */
function escapeHTML(s){
  return s.replace(/[&<>"']/g, c => ({
    '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'
  })[c]);
}

/* 辞書ロード（ローカルに失敗したら GitHub raw にフォールバック） */
async function loadDict(){
  const localPath = "シムネ語 詩日辞書.json";
  const rawUrl = `https://raw.githubusercontent.com/shimon314/simne-translator/5b02bfd1bcedece39d4cabe55e88aedebca7eb96/${encodeURIComponent("シムネ語 詩日辞書.json")}`;
  try{
    statusEl.textContent = "辞書ロード中… (ローカル)";
    await fetchAndBuild(localPath);
    ready = true;
    statusEl.textContent = `辞書ロード完了：${Object.keys(dict).length}語`;
  }catch(e1){
    try{
      statusEl.textContent = "ローカル辞書取得失敗、GitHub raw から取得中…";
      await fetchAndBuild(rawUrl);
      ready = true;
      statusEl.textContent = `辞書ロード完了 (raw)：${Object.keys(dict).length}語`;
    }catch(e2){
      ready = false;
      statusEl.textContent = "辞書ロード失敗";
      console.error(e1, e2);
    }
  }
}

async function fetchAndBuild(url){
  const res = await fetch(url, {cache:"no-store"});
  if(!res.ok) throw new Error("fetch failed");
  const json = await res.json();
  json.words.forEach(w=>{
    const sim = w.entry?.form;
    if(!sim) return;
    (w.translations||[]).forEach(t=>{
      if(!t.rawForms) return;
      t.rawForms.replace(/（.*?）|\(.*?\)/g,"")
      .split(/[、,]/).forEach(j=>{
        j=j.trim();
        if(j) dict[j]=sim;
      });
    });
  });
}

/* 翻訳本体 */
function translate(){
  if(!ready) return;
  const raw = inputEl.value;
  if(!raw || raw.trim()===""){
    outputEl.textContent = "";
    return;
  }
  const text = raw.trim();

  // コピュラ文 (例: 私は学生だ)
  let m = text.match(/^(.+?)は(.+?)(だ|です|である)$/);
  if(m){
    const subjJa = m[1];
    const predJa = m[2];
    const person = PRONOUN_PERSON[subjJa] || "third";
    const subj = dict[subjJa] || subjJa;
    const pred = dict[predJa] || predJa;
    outputEl.innerHTML = `${escapeHTML(subj)} <span class="tag">${escapeHTML(COPULA[person])}</span> ${escapeHTML(pred)}`;
    return;
  }

  // SVO文 (例: 私は世界を見る)
  m = text.match(/^(.+?)は(.+?)を(.+)$/);
  if(m){
    const subjJa = m[1];
    const objJa  = m[2];
    const verbJa = m[3];
    const subj = dict[subjJa] || subjJa;
    const obj  = dict[objJa]  || objJa;
    const verb = dict[verbJa] || verbJa;
    outputEl.innerHTML = `${escapeHTML(subj)} ${escapeHTML(verb)} ${escapeHTML(obj)}`;
    return;
  }

  // 部分・単語単位で辞書変換（未登録語はハイライト）
  const words = text.split(/\s+/);
  const out = words.map(w=>{
    // 句読点を保持
    const m = w.match(/^(.+?)([、,。\.！!？?]*)$/);
    const core = m ? m[1] : w;
    const tail = m ? m[2] : "";
    const sim = dict[core];
    if(sim){
      return `<span>${escapeHTML(sim)}</span>${escapeHTML(tail)}`;
    }else{
      // 未登録語はハイライトして元の日本語を表示
      return `<span class="unknown">${escapeHTML(core)}</span>${escapeHTML(tail)}`;
    }
  });
  outputEl.innerHTML = out.join(" ");
}
</script>

</body>
</html>
