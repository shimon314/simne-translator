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
</style>
</head>

<body>

<header>
  <b>Simne Translator</b>
  <div class="note" id="status">辞書ロード中…</div>
</header>

<main>

<div class="card">
  <textarea id="inputText" placeholder="私は学生だ / 私は世界を見る"></textarea>
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

/* 初期化 */
loadDict();
inputEl.addEventListener("input", translate);

/* 辞書ロード */
async function loadDict(){
  try{
    const res = await fetch("シムネ語 詩日辞書.json",{cache:"no-store"});
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
    ready=true;
    statusEl.textContent=`辞書ロード完了：${Object.keys(dict).length}語`;
  }catch{
    statusEl.textContent="辞書ロード失敗";
  }
}

/* 翻訳 */
function translate(){
  if(!ready) return;
  const text = inputEl.value.trim();
  outputEl.textContent="";
  if(!text) return;

  // コピュラ文 (例: 私は学生だ)
  let m = text.match(/^(.+?)は(.+?)(だ|です|である)$/);
  if(m){
    const subjJa = m[1];
    const predJa = m[2];
    const person = PRONOUN_PERSON[subjJa] || "third";
    const subj = dict[subjJa] || subjJa;
    const pred = dict[predJa] || predJa;
    outputEl.textContent = `${subj} ${COPULA[person]} ${pred}`;
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
    outputEl.textContent = `${subj} ${verb} ${obj}`;
    return;
  }

  // 単語単位で辞書変換
  const words = text.split(/\s+/);
  const out = words.map(w=>dict[w]||w);
  outputEl.textContent = out.join(" ");
}
</script>

</body>
</html>
