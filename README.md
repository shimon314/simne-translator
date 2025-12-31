<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Simne Translator (Stable Full) — Editable Dictionary</title>

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
  display:flex;
  align-items:center;
  justify-content:space-between;
}
.header-left{
  display:flex;
  flex-direction:column;
}
.controls{
  display:flex;
  gap:8px;
  align-items:center;
}
main{
  padding:12px;
  max-width:980px;
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
  resize:vertical;
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
  cursor:pointer;
  text-decoration:underline;
}
.tag{
  color:#60a5fa;
  font-weight:600;
}
.small{
  font-size:13px;
  color:#94a3b8;
}
button{
  background:#0b1220;
  color:#e5e7eb;
  border:1px solid #1e293b;
  padding:6px 10px;
  border-radius:8px;
  cursor:pointer;
}
button.secondary{
  background:transparent;
  border:1px dashed #334155;
}
.modal-backdrop{
  position:fixed;
  inset:0;
  background:rgba(0,0,0,0.6);
  display:flex;
  align-items:center;
  justify-content:center;
  z-index:50;
}
.modal{
  background:#041127;
  border:1px solid #1e293b;
  border-radius:10px;
  padding:16px;
  width:480px;
  max-width:92%;
  color:#e5e7eb;
}
.form-row{
  margin-bottom:8px;
  display:flex;
  gap:8px;
  align-items:center;
}
.form-row label{
  width:90px;
  font-size:13px;
  color:#94a3b8;
}
.form-row input, .form-row select, .form-row textarea{
  flex:1;
  background:transparent;
  border:1px solid #1e293b;
  padding:6px 8px;
  border-radius:6px;
  color:#e5e7eb;
  outline:none;
}
.small-muted{
  font-size:12px;
  color:#94a3b8;
}
.row{
  display:flex;
  gap:8px;
  align-items:center;
}
.file-input{
  display:inline-block;
  color:#e5e7eb;
}
</style>
</head>

<body>

<header>
  <div class="header-left">
    <b>Simne Translator — 編集対応版</b>
    <div class="note" id="status">辞書ロード中…</div>
  </div>
  <div class="controls">
    <label class="file-input">
      <input id="uploadJson" type="file" accept=".json" style="display:none">
      <button id="btnUpload" class="secondary">辞書アップロード</button>
    </label>
    <button id="btnDownload">JSONダウンロード</button>
    <button id="btnEditJson" class="secondary">JSON編集</button>
    <button id="btnReset" class="secondary">辞書リロード</button>
  </div>
</header>

<main>

<div class="card">
  <textarea id="inputText" placeholder="私は学生だ / 私は世界を見る" autocomplete="off" autocapitalize="off"></textarea>
  <div class="note">（入力中に即時翻訳、IME確定は待ちます。未登録語はクリックで追加できます）</div>
</div>

<div class="card">
  <div id="outputText" class="output"></div>
</div>

<div class="card">
  <div class="small-muted">辞書サイズ: <span id="dictSize">0</span> 語</div>
  <div class="small-muted" style="margin-top:8px">操作: 未登録語をクリック → 品詞 / シムネ語形 / 表示日本語 / 訳 を入力して追加できます。JSON編集で直接編集後「適用」で読み込みます。</div>
</div>

</main>

<!-- 追加モーダル -->
<div id="modalBackdrop" class="modal-backdrop" style="display:none">
  <div class="modal" role="dialog" aria-modal="true">
    <div style="font-weight:700;margin-bottom:8px">単語を辞書に追加</div>
    <div class="form-row">
      <label>未登録語</label>
      <input id="modalCore" disabled>
    </div>
    <div class="form-row">
      <label>品詞 (任意)</label>
      <select id="modalPOS">
        <option value="">— 指定しない —</option>
        <option value="noun">noun</option>
        <option value="verb">verb</option>
        <option value="adj">adjective</option>
        <option value="adv">adverb</option>
        <option value="pron">pronoun</option>
        <option value="other">other</option>
      </select>
    </div>
    <div class="form-row">
      <label>シムネ語</label>
      <input id="modalSim" placeholder="シムネ語の語形">
    </div>
    <div class="form-row">
      <label>表示日本語</label>
      <input id="modalJa" placeholder="画面に表示する日本語（省略可: 未登録語 をそのまま表示）">
    </div>
    <div class="form-row">
      <label>訳（複数はカンマ区切り）</label>
      <input id="modalTrans" placeholder="学生, 学ぶ人">
    </div>
    <div style="display:flex;gap:8px;justify-content:flex-end;margin-top:12px">
      <button id="modalCancel" class="secondary">キャンセル</button>
      <button id="modalAdd">追加して反映</button>
    </div>
  </div>
</div>

<!-- JSON編集モーダル -->
<div id="jsonBackdrop" class="modal-backdrop" style="display:none">
  <div class="modal" role="dialog" aria-modal="true">
    <div style="font-weight:700;margin-bottom:8px">辞書 JSON 編集</div>
    <textarea id="jsonEditor" style="min-height:240px;font-family:monospace"></textarea>
    <div style="display:flex;gap:8px;justify-content:flex-end;margin-top:8px">
      <button id="jsonCancel" class="secondary">閉じる</button>
      <button id="jsonApply">適用して読み込み</button>
    </div>
  </div>
</div>

<script>
const inputEl  = document.getElementById("inputText");
const outputEl = document.getElementById("outputText");
const statusEl = document.getElementById("status");
const dictSizeEl = document.getElementById("dictSize");
const uploadEl = document.getElementById("uploadJson");
const btnUpload = document.getElementById("btnUpload");
const btnDownload = document.getElementById("btnDownload");
const btnEditJson = document.getElementById("btnEditJson");
const btnReset = document.getElementById("btnReset");

const modalBackdrop = document.getElementById("modalBackdrop");
const modalCore = document.getElementById("modalCore");
const modalPOS = document.getElementById("modalPOS");
const modalSim = document.getElementById("modalSim");
const modalJa = document.getElementById("modalJa");
const modalTrans = document.getElementById("modalTrans");
const modalCancel = document.getElementById("modalCancel");
const modalAdd = document.getElementById("modalAdd");

const jsonBackdrop = document.getElementById("jsonBackdrop");
const jsonEditor = document.getElementById("jsonEditor");
const jsonCancel = document.getElementById("jsonCancel");
const jsonApply = document.getElementById("jsonApply");

/* コピュラ・人称 */
const PRONOUN_PERSON = {
  "私":"first","僕":"first","俺":"first",
  "あなた":"second","君":"second"
};
const COPULA = { first:"seg", second:"sem", third:"set" };

/* 辞書データ構造 */
let dict = {};          // 日本語表記 -> sim form
let jsonData = { words: [] }; // 元の JSON 全体を保持
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

document.addEventListener("click", (e)=>{
  // クリックで未登録語の追加モーダルを出す
  const t = e.target;
  if(t && t.dataset && t.dataset.unknownCore){
    openAddModal(t.dataset.unknownCore);
  }
});

btnUpload.addEventListener("click", ()=> uploadEl.click());
uploadEl.addEventListener("change", async (e)=>{
  const f = e.target.files[0];
  if(!f) return;
  try{
    const txt = await f.text();
    const parsed = JSON.parse(txt);
    jsonData = parsed;
    buildDictFromJson();
    statusEl.textContent = `アップロード辞書読み込み完了：${Object.keys(dict).length}語`;
    debounceTranslate();
  }catch(err){
    alert("JSONの読み込みに失敗しました: " + err.message);
  }
  uploadEl.value = "";
});

btnDownload.addEventListener("click", ()=>{
  const blob = new Blob([JSON.stringify(jsonData, null, 2)], {type:"application/json"});
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = "シムネ語 詩日辞書.json";
  document.body.appendChild(a);
  a.click();
  a.remove();
});

btnEditJson.addEventListener("click", ()=>{
  jsonEditor.value = JSON.stringify(jsonData, null, 2);
  jsonBackdrop.style.display = "flex";
});

jsonCancel.addEventListener("click", ()=>{ jsonBackdrop.style.display = "none"; });
jsonApply.addEventListener("click", ()=>{
  try{
    const parsed = JSON.parse(jsonEditor.value);
    jsonData = parsed;
    buildDictFromJson();
    jsonBackdrop.style.display = "none";
    statusEl.textContent = `手動編集を適用：${Object.keys(dict).length}語`;
    debounceTranslate();
  }catch(err){
    alert("JSON のパースに失敗しました: " + err.message);
  }
});

btnReset.addEventListener("click", ()=>{
  // リロードして初期辞書を再取得
  dict = {};
  ready = false;
  statusEl.textContent = "辞書をリロードしています…";
  loadDict(true);
});

/* モーダル操作 */
function openAddModal(core){
  modalCore.value = core;
  modalPOS.value = "";
  modalSim.value = "";
  modalJa.value = core;
  modalTrans.value = "";
  modalBackdrop.style.display = "flex";
}
modalCancel.addEventListener("click", ()=>{ modalBackdrop.style.display = "none"; });
modalAdd.addEventListener("click", ()=>{
  const core = modalCore.value.trim();
  const sim = modalSim.value.trim();
  const pos = modalPOS.value;
  const jaDisplay = modalJa.value.trim() || core;
  const transRaw = modalTrans.value.trim();

  if(!sim){
    alert("シムネ語の語形を入力してください。");
    return;
  }

  // translations として rawForms に日本語表記を入れる（カンマ区切り）
  const rawForms = transRaw || jaDisplay;
  addEntryToJson({
    entry: { form: sim },
    translations: [{ pos: pos || undefined, rawForms: rawForms }]
  });
  modalBackdrop.style.display = "none";
  debounceTranslate();
});

/* エスケープ（安全に innerHTML にするため） */
function escapeHTML(s){
  return String(s).replace(/[&<>"']/g, c => ({
    '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'
  })[c]);
}

/* 辞書ロード（ローカルに失敗したら GitHub raw にフォールバック） */
async function loadDict(forceRaw=false){
  const localPath = "シムネ語 詩日辞書.json";
  const rawUrl = `https://raw.githubusercontent.com/shimon314/simne-translator/5b02bfd1bcedece39d4cabe55e88aedebca7eb96/${encodeURIComponent("シムネ語 詩日辞書.json")}`;
  try{
    statusEl.textContent = "辞書ロード中… (ローカル)";
    if(forceRaw){
      throw new Error("force raw");
    }
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
  jsonData = json;
  buildDictFromJson();
}

function buildDictFromJson(){
  dict = {};
  (jsonData.words || []).forEach(w=>{
    const sim = w.entry?.form;
    if(!sim) return;
    (w.translations||[]).forEach(t=>{
      if(!t.rawForms) return;
      // remove parentheses, then split
      t.rawForms.replace(/（.*?）|\(.*?\)/g,"")
      .split(/[、,]/).forEach(j=>{
        j=j.trim();
        if(j) dict[j]=sim;
      });
    });
  });
  dictSizeEl.textContent = Object.keys(dict).length;
}

/* 辞書に項目を追加する（jsonData に push して dict を更新） */
function addEntryToJson(item){
  // item expected: { entry: {form}, translations: [{pos, rawForms}] }
  if(!jsonData.words) jsonData.words = [];
  jsonData.words.push(item);
  // rebuild small part of dict from item
  const sim = item.entry?.form;
  (item.translations||[]).forEach(t=>{
    if(!t.rawForms) return;
    t.rawForms.replace(/（.*?）|\(.*?\)/g,"")
    .split(/[、,]/).forEach(j=>{
      j=j.trim();
      if(j) dict[j]=sim;
    });
  });
  dictSizeEl.textContent = Object.keys(dict).length;
  statusEl.textContent = `辞書に追加：${Object.keys(dict).length}語`;
}

/* 翻訳本体（未登録語は推測翻訳なし。クリックで追加） */
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
    const pred = dict[predJa];
    const predHtml = pred ? escapeHTML(pred) : `<span class="unknown" data-unknown-core="${escapeHTML(predJa)}">${escapeHTML(predJa)}</span>`;
    outputEl.innerHTML = `${escapeHTML(subj)} <span class="tag">${escapeHTML(COPULA[person])}</span> ${predHtml}`;
    attachUnknownDataset();
    return;
  }

  // SVO文 (例: 私は世界を見る)
  m = text.match(/^(.+?)は(.+?)を(.+)$/);
  if(m){
    const subjJa = m[1];
    const objJa  = m[2];
    const verbJa = m[3];
    const subj = dict[subjJa] || subjJa;
    const obj  = dict[objJa];
    const verb = dict[verbJa];
    const objHtml = obj ? escapeHTML(obj) : `<span class="unknown" data-unknown-core="${escapeHTML(objJa)}">${escapeHTML(objJa)}</span>`;
    const verbHtml = verb ? escapeHTML(verb) : `<span class="unknown" data-unknown-core="${escapeHTML(verbJa)}">${escapeHTML(verbJa)}</span>`;
    outputEl.innerHTML = `${escapeHTML(subj)} ${verbHtml} ${objHtml}`;
    attachUnknownDataset();
    return;
  }

  // 単語単位で辞書変換（未登録語はハイライト、かつクリックで追加）
  const words = text.split(/\s+/);
  const out = words.map(w=>{
    // 句読点を保持
    const mm = w.match(/^(.+?)([、,。\.！!？?]*)$/);
    const core = mm ? mm[1] : w;
    const tail = mm ? mm[2] : "";
    const sim = dict[core];
    if(sim){
      return `<span>${escapeHTML(sim)}</span>${escapeHTML(tail)}`;
    }else{
      // 未登録語はハイライトして、data 属性でクリックで追加できるようにする
      return `<span class="unknown" data-unknown-core="${escapeHTML(core)}">${escapeHTML(core)}</span>${escapeHTML(tail)}`;
    }
  });
  outputEl.innerHTML = out.join(" ");
  attachUnknownDataset();
}

/* attachUnknownDataset: ensure dataset attributes are accessible from DOM (some browsers decode) */
function attachUnknownDataset(){
  // re-set dataset explicitly for unknown spans (to be safe)
  const spans = outputEl.querySelectorAll('.unknown');
  spans.forEach(s=>{
    const text = s.textContent || s.innerText || "";
    // If text contains punctuation because of earlier encoding, strip trailing punctuation for core
    const core = text.replace(/[、,。\.！!？?]+$/g, "");
    s.dataset.unknownCore = core;
  });
}

/* openAddModal when clicking unknown: handled by document click listener */

/* デバウンス */
function debounceTranslate(){
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(translate, DEBOUNCE_MS);
}

</script>

</body>
</html>
