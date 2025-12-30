<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>日本語 → シムネ語 翻訳機</title>

<style>
body {
  background:#0f1115;
  color:#eaeaea;
  font-family: system-ui, sans-serif;
  padding:20px;
}
h1 { font-size:22px; }
textarea, input {
  width:100%;
  background:#1c1f26;
  color:#fff;
  border:1px solid #444;
  padding:10px;
  margin-bottom:10px;
}
button {
  padding:10px 16px;
  font-size:15px;
  cursor:pointer;
}
#output span.unknown {
  color:#ff6b6b;
  text-decoration: underline;
}
#status {
  font-size:13px;
  opacity:0.8;
  margin-bottom:10px;
}
#dictSearchResult {
  background:#1c1f26;
  padding:8px;
  max-height:120px;
  overflow:auto;
  font-size:14px;
}
</style>
</head>

<body>

<h1>日本語 → シムネ語 翻訳機</h1>

<div id="status">辞書未ロード</div>

<input id="dictUrl" value="シムネ語 詩日辞書.json">

<textarea id="input" placeholder="例：私 は 世界 を 見る"></textarea>

<button onclick="translate()">翻訳</button>

<h3>翻訳結果</h3>
<div id="output"></div>

<hr>

<h3>辞書検索（日本語）</h3>
<input id="dictSearch" placeholder="例：私 / 世界 / 見る" oninput="searchDict()">
<div id="dictSearchResult"></div>

<script>
let jaToSimne = {};
let loaded = false;

async function loadDictionary(url) {
  if (loaded) return;

  const res = await fetch(url);
  const data = await res.json();

  for (const word of data.words) {
    const simne = word.entry?.form;
    if (!simne) continue;

    for (const tr of word.translations ?? []) {
      if (!tr.rawForms) continue;

      const cleaned = tr.rawForms
        .replace(/（.*?）|\(.*?\)/g, "")
        .split(/[、,]/);

      for (const ja of cleaned) {
        const key = ja.trim();
        if (key) jaToSimne[key] = simne;
      }
    }
  }

  loaded = true;
  document.getElementById("status").textContent =
    `辞書ロード完了：${Object.keys(jaToSimne).length} 語`;
}

function t(word) {
  return jaToSimne[word] ?? null;
}

async function translate() {
  const url = document.getElementById("dictUrl").value;
  await loadDictionary(url);

  const words = document.getElementById("input").value.trim().split(/\s+/);

  let S=[], O=[], V=[], mode="S";

  for (const w of words) {
    if (["は","が"].includes(w)) { mode="O"; continue; }
    if (w==="を") { mode="V"; continue; }

    if (mode==="S") S.push(w);
    else if (mode==="O") O.push(w);
    else V.push(w);
  }

  const ordered = [...S, ...V, ...O];

  const out = ordered.map(w=>{
    const r = t(w);
    return r ? r : `<span class="unknown">${w}</span>`;
  }).join(" ");

  document.getElementById("output").innerHTML = out;
}

function searchDict() {
  const q = document.getElementById("dictSearch").value.trim();
  if (!q) {
    document.getElementById("dictSearchResult").textContent="";
    return;
  }
  const r = jaToSimne[q];
  document.getElementById("dictSearchResult").textContent =
    r ? `${q} → ${r}` : "未登録";
}
</script>

</body>
</html>
