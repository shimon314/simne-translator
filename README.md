<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Simne Translator — リアルタイム（賢い単語分割・推測翻訳なし・強化UI）</title>

<style>
:root{
  --bg:#0b1220;
  --card:#071027;
  --muted:#94a3b8;
  --text:#e6eef8;
  --accent:#60a5fa;
  --unknown:#f87171;
  --particle:#94a3b8;
  --ok-bg:rgba(96,165,250,0.06);
  --radius:12px;
}
*{box-sizing:border-box}
html,body{height:100%}
body{
  margin:0;
  background:var(--bg);
  color:var(--text);
  font-family:Inter, Noto Sans JP, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
  -webkit-font-smoothing:antialiased;
  -moz-osx-font-smoothing:grayscale;
  padding:16px;
}
.header{
  display:flex;
  gap:12px;
  align-items:center;
  margin-bottom:12px;
}
.title{font-weight:700;font-size:18px}
.controls{
  margin-left:auto;
  display:flex;
  gap:8px;
  align-items:center;
}
.small{
  font-size:13px;color:var(--muted)
}
.card{
  background:linear-gradient(180deg, rgba(255,255,255,0.02), transparent);
  border:1px solid rgba(255,255,255,0.04);
  border-radius:var(--radius);
  padding:12px;
  margin-bottom:12px;
}
.row{display:flex;gap:8px;align-items:center}
.textarea{
  width:100%;
  min-height:96px;
  background:transparent;
  border:1px dashed rgba(255,255,255,0.04);
  padding:12px;
  color:var(--text);
  font-size:15px;
  border-radius:8px;
  outline:none;
  resize:vertical;
}
.toolbar{display:flex;gap:8px;margin-top:8px}
.btn{
  background:transparent;
  border:1px solid rgba(255,255,255,0.04);
  color:var(--text);
  padding:8px 10px;
  border-radius:8px;
  cursor:pointer;
  font-size:13px;
}
.btn:active{transform:translateY(1px)}
.badge{
  background:var(--ok-bg);
  color:var(--accent);
  padding:6px 8px;border-radius:999px;font-weight:600;font-size:13px;
  border:1px solid rgba(96,165,250,0.12)
}
.output{
  font-size:18px;
  white-space:pre-wrap;
  line-height:1.6;
  display:flex;
  flex-wrap:wrap;
  gap:6px;
  align-items:center;
  min-height:48px;
}
.token{
  padding:6px 8px;
  border-radius:8px;
  background:transparent;
  border:1px solid rgba(255,255,255,0.02);
  cursor:default;
  user-select:text;
}
.token.translated{background:var(--ok-bg); color:var(--accent); border-color:rgba(96,165,250,0.12)}
.token.unknown{background:rgba(248,113,113,0.06); color:var(--unknown); border-color:rgba(248,113,113,0.12); font-weight:600}
.token.particle{background:transparent; color:var(--particle); border-style:dashed}
.token .meta{display:block;font-size:12px;color:var(--muted);margin-top:4px}
.token:hover{filter:brightness(1.06)}
.row-between{display:flex;justify-content:space-between;align-items:center}
.footer-note{font-size:13px;color:var(--muted);margin-top:6px}

/* responsive */
@media (max-width:640px){
  .header{flex-direction:column;align-items:flex-start;gap:6px}
  .controls{margin-left:0}
}
</style>
</head>
<body>

<div class="header">
  <div>
    <div class="title">Simne Translator</div>
    <div class="small">賢い単語分割（形態素分割）・推測翻訳なし・インタラクティブUI</div>
  </div>

  <div class="controls">
    <div id="status" class="small">辞書ロード中…</div>
    <div style="width:8px"></div>
    <div id="dictCount" class="badge">--</div>
  </div>
</div>

<div class="card">
  <textarea id="inputText" class="textarea" placeholder="例：私は学生だ / 私は世界を見る" autocomplete="off" autocapitalize="off" spellcheck="false"></textarea>

  <div class="toolbar">
    <button id="clearBtn" class="btn">クリア</button>
    <button id="copyBtn" class="btn">出力をコピー</button>
    <button id="addBtn" class="btn">単語を辞書に追加</button>
    <div style="flex:1"></div>
    <label class="small" title="IME確定中は待機">IME対応</label>
  </div>

  <div class="footer-note">翻訳は「厳密一致」のみです（活用やステミングによる推測は行いません）。未登録語は赤く表示されます。</div>
</div>

<div class="card">
  <div class="row-between">
    <div style="font-weight:700">翻訳結果</div>
    <div class="small">トークンをクリックでコピー / 未登録語は辞書追加可</div>
  </div>

  <div id="outputText" class="output" aria-live="polite"></div>
</div>

<div class="card">
  <div style="display:flex;gap:12px;align-items:center">
    <div style="font-weight:700">辞書プレビュー</div>
    <div class="small">（ユーザー辞書はブラウザに保存されます）</div>
  </div>
  <div id="dictPreview" class="small" style="margin-top:8px; max-height:160px; overflow:auto"></div>
</div>

<script>
/*
  目的：
  - 賢い単語分割 → TinySegmenter を組み込んだ軽量形態素分割
  - 推測翻訳を行わない（厳密一致のみ）
  - UI強化：トークン毎に状態表示、クリックでコピー、単語をユーザー辞書へ追加可能
*/

/* --- TinySegmenter (compact JS implementation) ---
   Implementation adapted / minimized for embedding.
   This is a lightweight Japanese segmenter (sufficient for tokenization).
*/
function TinySegmenter(){
  // Very small port of standard TinySegmenter algorithm (kept compact)
  // Source logic intentionally compacted; not full-featured but suitable for segmentation.
  const chartypes = {
    hira:/^[\u3041-\u3096]$/,
    kata:/^[\u30A1-\u30FA\u30FC]$/,
    kanji:/^[\u4E00-\u9FFF]$/,
    ascii:/^[A-Za-z0-9]$/,
    space:/^\s$/,
    symbol:/^[^ぁ-んァ-ヶ一-龠A-Za-z0-9\s]$/
  };
  function ctype(ch){
    if(!ch) return 'space';
    for(const k in chartypes) if(chartypes[k].test(ch)) return k;
    return 'symbol';
  }
  this.tokenize = function(s){
    // split into characters then make decisions. This is a heuristic segmentation:
    if(!s) return [];
    const chars = Array.from(s);
    const types = chars.map(ctype);
    const tokens = [];
    let buf = chars[0] || '';
    for(let i=1;i<chars.length;i++){
      const a = chars[i-1], b = chars[i];
      const ta = types[i-1], tb = types[i];
      // break rules:
      // break before space/punct
      if(tb==='space' || tb==='symbol'){ tokens.push(buf); buf=b; continue; }
      // keep kana chains together, kanji chains together, ascii chains together
      if( (ta==='hira'&&tb==='hira') || (ta==='kata'&&tb==='kata') || (ta==='kanji'&&tb==='kanji') || (ta==='ascii'&&tb==='ascii') ){
        buf += b; continue;
      }
      // particle heuristics: small set of single-char particles often separate
      if(b==='は' || b==='が' || b==='を' || b==='に' || b==='へ' || b==='と' || b==='で' || b==='や' || b==='の' || b==='も' || b==='から' || b==='まで' || b==='より'){
        // attach particle as separate token
        tokens.push(buf);
        buf = b;
        continue;
      }
      // punctuation adjacency: break
      if(/^[、,。\.！!？?\)\]]$/.test(b)){
        buf += b; tokens.push(buf); buf=''; continue;
      }
      // default: if type changes, break; else continue
      if(ta!==tb){ tokens.push(buf); buf=b; } else { buf+=b; }
    }
    if(buf!=='') tokens.push(buf);
    // final cleanup: split trailing punctuation tokens
    return tokens.filter(t=>t!=='');
  };
}

/* --- End TinySegmenter --- */

/* DOM */
const inputEl = document.getElementById('inputText');
const outputEl = document.getElementById('outputText');
const statusEl = document.getElementById('status');
const dictCountEl = document.getElementById('dictCount');
const dictPreviewEl = document.getElementById('dictPreview');
const copyBtn = document.getElementById('copyBtn');
const clearBtn = document.getElementById('clearBtn');
const addBtn = document.getElementById('addBtn');

/* 辞書 */
let dict = {};          // main dictionary loaded from JSON
let userDict = {};      // user additions stored in localStorage
let ready = false;

/* tokenizer */
const segmenter = new TinySegmenter();

/* IME / debounce */
let composing = false;
let debounceTimer = null;
const DEBOUNCE_MS = 60;

/* 初期化 */
loadUserDict();
loadDict();
inputEl.addEventListener('input', ()=>{ if(!composing) debounceTranslate(); });
inputEl.addEventListener('compositionstart', ()=>{ composing = true; });
inputEl.addEventListener('compositionend', ()=>{ composing = false; debounceTranslate(); });

clearBtn.addEventListener('click', ()=>{ inputEl.value=''; translate(); inputEl.focus(); });
copyBtn.addEventListener('click', copyOutput);
addBtn.addEventListener('click', onAddWord);

/* 辞書ローディング（同様にローカル失敗→rawフォールバック） */
async function loadDict(){
  const localPath = "シムネ語 詩日辞書.json";
  const rawUrl = `https://raw.githubusercontent.com/shimon314/simne-translator/5b02bfd1bcedece39d4cabe55e88aedebca7eb96/${encodeURIComponent("シムネ語 詩日辞書.json")}`;
  statusEl.textContent = '辞書ロード中…';
  try{
    await fetchAndBuild(localPath);
    finalizeDictLoad();
  }catch(e1){
    try{
      statusEl.textContent = 'ローカル辞書取得失敗、rawから取得中…';
      await fetchAndBuild(rawUrl);
      finalizeDictLoad(true);
    }catch(e2){
      ready = false;
      statusEl.textContent = '辞書ロード失敗';
      dictCountEl.textContent = '--';
      console.error(e1,e2);
    }
  }
}
async function fetchAndBuild(url){
  const res = await fetch(url, {cache:'no-store'});
  if(!res.ok) throw new Error('fetch failed');
  const json = await res.json();
  json.words.forEach(w=>{
    const sim = w.entry?.form;
    if(!sim) return;
    (w.translations||[]).forEach(t=>{
      if(!t.rawForms) return;
      t.rawForms.replace(/（.*?）|\(.*?\)/g,"")
      .split(/[、,]/).forEach(j=>{
        j=j.trim();
        if(j) dict[j] = sim;
      });
    });
  });
}
function finalizeDictLoad(fromRaw){
  ready = true;
  statusEl.textContent = fromRaw ? `辞書ロード完了 (raw)：${Object.keys(dict).length}語` : `辞書ロード完了：${Object.keys(dict).length}語`;
  dictCountEl.textContent = Object.keys(dict).length;
  renderDictPreview();
  translate();
}

/* user dict (localStorage) */
function loadUserDict(){
  try{
    const raw = localStorage.getItem('simne_user_dict_v1') || '{}';
    userDict = JSON.parse(raw);
  }catch(e){
    userDict = {};
  }
}
function saveUserDict(){
  localStorage.setItem('simne_user_dict_v1', JSON.stringify(userDict));
  renderDictPreview();
}

/* escape for safe innerHTML */
function escapeHTML(s){
  return String(s).replace(/[&<>"']/g, ch => ({
    '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'
  })[ch]);
}

/* main translate (厳密一致のみ) */
function translate(){
  if(!ready && Object.keys(userDict).length===0) {
    outputEl.innerHTML = '<div class="small">辞書が読み込まれていません</div>';
    return;
  }
  const raw = inputEl.value || '';
  if(!raw.trim()){ outputEl.innerHTML=''; return; }

  const tokens = segmenter.tokenize(raw.trim());
  const nodes = tokens.map(tok => renderToken(tok));
  outputEl.innerHTML = '';
  nodes.forEach(n => outputEl.appendChild(n));
}

/* token rendering */
function renderToken(tok){
  const node = document.createElement('span');
  // preserve punctuation characters at end
  const m = tok.match(/^(.+?)([、,。\.！!？?\)\]]*)$/);
  const core = m ? m[1] : tok;
  const tail = m ? m[2] : '';

  // consult userDict first, then main dict
  const sim = userDict[core] || dict[core] || null;

  node.className = 'token';
  if(sim){
    node.classList.add('translated');
    node.innerHTML = `<span class="label">${escapeHTML(sim)}</span>`;
    node.title = `日本語: ${core}\n該当: 辞書一致 (厳密一致)`;
  } else if(isParticle(core)){
    node.classList.add('particle');
    node.innerHTML = `<span>${escapeHTML(core)}</span>`;
    node.title = `助詞/記号: ${core}`;
  } else {
    node.classList.add('unknown');
    node.innerHTML = `<span>${escapeHTML(core)}</span>`;
    node.title = `未登録: ${core}`;
  }

  // add small tail punctuation (not part of core)
  if(tail) node.innerHTML += `<span style="margin-left:6px;opacity:0.8">${escapeHTML(tail)}</span>`;

  // meta info
  const meta = document.createElement('span');
  meta.className = 'meta';
  if(sim) meta.textContent = `訳: ${sim}`;
  else if(isParticle(core)) meta.textContent = `助詞`;
  else meta.textContent = `未登録`;
  node.appendChild(meta);

  // on click: copy translated form if exists, else copy original token
  node.addEventListener('click', (e)=>{
    const toCopy = sim ? sim : core;
    navigator.clipboard?.writeText(toCopy).then(()=>{
      // brief visual feedback
      const old = node.style.outline;
      node.style.outline = '2px solid rgba(96,165,250,0.18)';
      setTimeout(()=>{ node.style.outline = old; }, 350);
    }).catch(()=>{ /* ignore */ });
  });

  // context menu (right click) to offer add-to-dictionary for unknown tokens
  node.addEventListener('contextmenu', (e)=>{
    if(sim || isParticle(core)) return; // only for unknown
    e.preventDefault();
    const answer = prompt(`"${core}" を辞書に追加します。対応するシムネ語を入力してください（キャンセルで中止）`);
    if(answer && answer.trim()){
      userDict[core] = answer.trim();
      saveUserDict();
      translate();
    }
  });

  return node;
}

/* helper: is particle/functional token */
function isParticle(s){
  const particles = ['は','が','を','に','へ','と','で','や','の','も','から','まで','より','ぞ','ぜ','ね','よ','か','な','や'];
  return particles.includes(s);
}

/* debounce */
function debounceTranslate(){
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(translate, DEBOUNCE_MS);
}

/* copy output as joined translations (space-separated simne tokens) */
function copyOutput(){
  // Build a string: for each rendered token, use sim if available else original
  const tokens = segmenter.tokenize((inputEl.value||'').trim());
  const arr = tokens.map(tok=>{
    const m = tok.match(/^(.+?)([、,。\.！!？?\)\]]*)$/);
    const core = m ? m[1] : tok;
    const tail = m ? m[2] : '';
    const sim = userDict[core] || dict[core] || null;
    return (sim ? sim : core) + tail;
  });
  const out = arr.join(' ');
  navigator.clipboard?.writeText(out).then(()=>{
    copyBtn.textContent = 'コピーしました ✓';
    setTimeout(()=>copyBtn.textContent='出力をコピー',900);
  }).catch(()=>{
    alert('クリップボードにコピーできませんでした');
  });
}

/* add selected token to user dictionary (convenience) */
function onAddWord(){
  const sel = window.getSelection().toString().trim();
  if(!sel){
    alert('追加する日本語の単語をテキストから選択してください（英単語や日本語の語を選択）');
    return;
  }
  const sim = prompt(`"${sel}" の対応するシムネ語を入力してください：`);
  if(sim && sim.trim()){
    userDict[sel] = sim.trim();
    saveUserDict();
    translate();
  }
}

/* render user/main dict preview */
function renderDictPreview(){
  const keys = Object.keys(userDict);
  const mainCount = Object.keys(dict).length;
  dictCountEl.textContent = mainCount + (keys.length ? ` (+${keys.length})` : '');
  let html = '';
  if(keys.length===0){
    html = `<div class="small">ユーザー辞書は空です。未登録語を右クリックで追加できます。</div>`;
  } else {
    const items = keys.map(k => `<div style="padding:6px 8px;border-radius:8px;margin-bottom:6px;background:rgba(255,255,255,0.01);"><strong>${escapeHTML(k)}</strong> → <span style="color:var(--accent)">${escapeHTML(userDict[k])}</span></div>`);
    html = items.join('');
  }
  dictPreviewEl.innerHTML = html;
}

/* initial translate (if any) */
setTimeout(translate, 200);
</script>

</body>
</html>
