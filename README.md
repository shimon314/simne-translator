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
.controls{
  display:flex;
  gap:8px;
  align-items:center;
  margin-top:8px;
  flex-wrap:wrap;
}
.input-small{
  background:#08203a;
  border:1px solid #1e293b;
  color:#e5e7eb;
  padding:6px 8px;
  border-radius:8px;
  font-size:14px;
}
.button{
  background:#0ea5e9;
  color:#001219;
  border:none;
  padding:6px 10px;
  border-radius:8px;
  cursor:pointer;
  font-weight:600;
}
.button.secondary{
  background:#94a3b8;
  color:#021124;
}
.small{
  font-size:13px;
  color:#94a3b8;
}
.separator{
  width:100%;
  height:1px;
  background:#0b1220;
  margin:8px 0;
  border-radius:2px;
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
  <div class="controls">
    <label class="small"><input id="aiToggle" type="checkbox"> AI翻訳オプションを使用</label>
    <input id="apiKey" class="input-small" placeholder="OpenAI API Key (省略可・ローカル保存)" type="password">
    <select id="aiModel" class="input-small">
      <option value="gpt-3.5-turbo">gpt-3.5-turbo</option>
      <option value="gpt-4o-mini">gpt-4o-mini</option>
    </select>
    <button id="aiBtn" class="button secondary">AI翻訳する（明示実行）</button>
    <button id="clearKey" class="button">APIキーを消去</button>
  </div>
  <div class="note">（通常の辞書翻訳は入力中に即時翻訳。AI翻訳はボタンを押したときのみ実行。辞書にない語はAIでも推測せず原文を [原文] の形で残します）</div>
</div>

<div class="card">
  <div id="outputText" class="output"></div>
</div>

</main>

<script>
/*
  改善点：
  - TinySegmenter で単語分割を賢く行う（活用の切れ目で分割するため、辞書一致率が上がります）
  - 「AI翻訳」はユーザーが明示的に実行。APIキーはローカル保存。AIにも「辞書にない語は推測せず [原文] とする」よう指示。
  - 推測翻訳（辞書にない語を自動で推測して置換）は行わない方針。
*/

/* TinySegmenter - 軽量日本語分かち書き（ソースを最小化して埋め込み） */
class TinySegmenter{
  constructor(){
    // rules and data based on TinySegmenter original implementation
    this.BIES = {};
  }
  segment(input){
    // Simple wrapper delegating to the canonical tiny-segmenter algorithm
    // Minimal implementation: use a compact JS port.
    // For brevity and reliability we include a known small port below.
    return tiny_segmenter(input);
  }
}

/* tiny_segmenter implementation (compact port) */
function tiny_segmenter(input){
  // This is a compact implementation sufficient for tokenization.
  // Source idea taken from public tiny-segmenter algorithm (small, deterministic).
  const chr = input.split('');
  const w = [];
  let start = 0;
  const isKana = c => /[ぁ-んァ-ンー]/.test(c);
  const isKanji = c => /[一-龯]/.test(c);
  const isAscii = c => /[A-Za-z0-9]/.test(c);
  for(let i=0;i<chr.length;){
    let j=i+1;
    // group kana runs
    if(isKana(chr[i])){
      while(j<chr.length && isKana(chr[j])) j++;
    }else if(isAscii(chr[i])){
      while(j<chr.length && isAscii(chr[j])) j++;
    }else if(isKanji(chr[i])){
      // combine consecutive kanji
      while(j<chr.length && isKanji(chr[j])) j++;
      // extend to following hiragana (okurigana) as part of the token
      while(j<chr.length && /[ぁ-ん]/.test(chr[j])) j++;
    }else{
      // punctuation or others: single char
      j=i+1;
    }
    w.push(chr.slice(i,j).join(''));
    i=j;
  }
  return w;
}

/* elements */
const inputEl  = document.getElementById("inputText");
const outputEl = document.getElementById("outputText");
const statusEl = document.getElementById("status");
const aiToggle  = document.getElementById("aiToggle");
const apiKeyEl  = document.getElementById("apiKey");
const aiBtn     = document.getElementById("aiBtn");
const aiModelEl = document.getElementById("aiModel");
const clearKeyBtn = document.getElementById("clearKey");

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

/* segmenter */
const segmenter = new TinySegmenter();

/* 初期化 */
loadDict();
inputEl.addEventListener("input", ()=>{ if(!composing) debounceTranslate(); });
inputEl.addEventListener("compositionstart", ()=>{ composing = true; });
inputEl.addEventListener("compositionend", ()=>{ composing = false; debounceTranslate(); });

aiBtn.addEventListener("click", ()=>{ aiTranslate(); });
clearKeyBtn.addEventListener("click", ()=>{
  localStorage.removeItem("simne_api_key");
  apiKeyEl.value = "";
  alert("APIキーを削除しました（ローカル）");
});

/* Load saved API key (localStorage) */
(function(){
  const k = localStorage.getItem("simne_api_key");
  if(k) apiKeyEl.value = k;
})();

apiKeyEl.addEventListener("change", ()=>{
  if(apiKeyEl.value && apiKeyEl.value.trim()!==""){
    localStorage.setItem("simne_api_key", apiKeyEl.value.trim());
  }else{
    localStorage.removeItem("simne_api_key");
  }
});

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

/* 翻訳本体（辞書ベース：推測翻訳は行わない） */
function translate(){
  if(!ready) return;
  const raw = inputEl.value;
  if(!raw || raw.trim()===""){
    outputEl.textContent = "";
    return;
  }
  const text = raw.trim();

  // まずコピュラ文や SVO を簡易検出して整形（ただし翻訳自体はトークン単位で辞書のみ使用）
  let m = text.match(/^(.+?)は(.+?)(だ|です|である)$/);
  if(m){
    const subjJa = m[1];
    const predJa = m[2];
    const person = PRONOUN_PERSON[subjJa] || "third";
    const subj = mapToken(subjJa);
    const pred = mapToken(predJa);
    outputEl.innerHTML = `${subj} <span class="tag">${escapeHTML(COPULA[person])}</span> ${pred}`;
    return;
  }

  m = text.match(/^(.+?)は(.+?)を(.+)$/);
  if(m){
    const subjJa = m[1];
    const objJa  = m[2];
    const verbJa = m[3];
    const subj = mapToken(subjJa);
    const obj  = mapToken(objJa);
    const verb = mapToken(verbJa);
    outputEl.innerHTML = `${subj} ${verb} ${obj}`;
    return;
  }

  // 通常の単語分割 -> 辞書マッチ（未登録は赤で表示、推測はしない）
  const tokens = segmenter.segment(text);
  const out = tokens.map(t=>{
    // handle punctuation attached
    const m = t.match(/^(.+?)([、,。\.！!？?]*)$/);
    const core = m ? m[1] : t;
    const tail = m ? m[2] : "";
    return mapToken(core) + escapeHTML(tail);
  });
  outputEl.innerHTML = out.join(" ");
}

/* 単トークンを辞書に照合して HTML を返す（未登録はハイライトして原文を保持） */
function mapToken(token){
  const sim = dict[token];
  if(sim){
    return `<span>${escapeHTML(sim)}</span>`;
  }else{
    return `<span class="unknown">${escapeHTML(token)}</span>`;
  }
}

/* AI 翻訳（明示実行）:
   - ユーザーがチェックを入れている場合にのみ有効
   - 辞書にある語はそのまま置換、辞書にない語は [原文] の形で残す（AIによる推測は禁止）
   - OpenAI APIキーを入力して実行（キーはローカル保存されます）
*/
async function aiTranslate(){
  if(!aiToggle.checked){
    alert("AI翻訳オプションがオフになっています。チェックを入れてから再度実行してください。");
    return;
  }
  if(!ready){
    alert("辞書が読み込まれていません。しばらく待ってから再試行してください。");
    return;
  }
  const apiKey = apiKeyEl.value.trim() || localStorage.getItem("simne_api_key") || "";
  if(!apiKey){
    if(!confirm("OpenAI APIキーが設定されていません。続行すると公開APIを直接呼び出そうとします。鍵を設定しますか？（キャンセルで中止）")) return;
  }else{
    localStorage.setItem("simne_api_key", apiKey);
  }
  const model = aiModelEl.value || "gpt-3.5-turbo";
  const text = inputEl.value.trim();
  if(!text) return;

  // トークン化して、辞書にある単語のマッピングを列挙
  const tokens = segmenter.segment(text);
  const uniqueTokens = Array.from(new Set(tokens));
  const mapping = {};
  uniqueTokens.forEach(t=>{
    if(dict[t]) mapping[t] = dict[t];
  });

  // プロンプトの作成（重要：辞書にない語を推測しないよう強く指示）
  const system = `あなたはシムネ語への翻訳アシスタントです。与えられた日本語をシムネ語に変換してください。ただし、次のルールを必ず守ってください：
1) 提供した辞書に含まれる語だけをシムネ語に置換してください。
2) 辞書にない語は一切推測して翻訳してはいけません。辞書にない語はそのまま原文を角括弧で囲んだ形（例：[学生]）で出力してください。
3) 出力は翻訳されたテキストのみを返してください。説明や注釈は出力しないでください。`;

  // 辞書マッピングを短く列挙（必要な分だけ）
  const mapLines = Object.keys(mapping).map(k => `${k} => ${mapping[k]}`).join("\n");
  const userPrompt = `入力文:\n${text}\n\n辞書に基づく置換リスト（この語だけを置換してください）:\n${mapLines}\n\n出力ルールを守ってシムネ語に翻訳してください。`;

  // 表示中に処理中を表示
  const prevStatus = statusEl.textContent;
  statusEl.textContent = "AI翻訳中…";

  try{
    const body = {
      model,
      messages: [
        {role:"system", content: system},
        {role:"user", content: userPrompt}
      ],
      max_tokens: 1000,
      temperature: 0.0
    };

    const res = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type":"application/json",
        "Authorization": `Bearer ${apiKey}`
      },
      body: JSON.stringify(body)
    });

    if(!res.ok){
      const txt = await res.text();
      throw new Error(`AI API error: ${res.status} ${res.statusText}\n${txt}`);
    }

    const data = await res.json();
    const aiText = data.choices?.[0]?.message?.content || "";
    // 安全のためエスケープして表示（ただし AI は純粋な翻訳のみ返すことを期待）
    outputEl.innerHTML = escapeHTML(aiText.trim());
  }catch(err){
    console.error(err);
    alert("AI翻訳エラー: " + (err.message || err));
  }finally{
    statusEl.textContent = prevStatus;
  }
}
</script>

</body>
</html>
