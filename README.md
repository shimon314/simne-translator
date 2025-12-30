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
}
</style>
</head>

<body>

<h1>日本語 → シムネ語 翻訳機</h1>

<div id="status">辞書未ロード</div>

<textarea id="input" placeholder="例：私は世界を見る"></textarea>
<button onclick="translate()">翻訳</button>

<h3>翻訳結果</h3>
<div id="output"></div>

<hr>

<h3>辞書検索（日本語）</h3>
<input id="dictSearch" placeholder="例：私 / 世界 / 見る">
<div id="dictSearchResult"></div>

<script>
let jaToSimne = {};
let dictLoaded = false;

/* 辞書ロード */
async function loadDictionary() {
  if (dictLoaded) return;

  const res = await fetch("シムネ語 詩日辞書.json");
  const data = await res.json();

  for (const word of data.words) {
    const simne = word.entry?.form;
    if (!simne) continue;

    for (const tr of word.translations ?? []) {
      if (!tr.rawForms) continue;

      const list = tr.rawForms
        .replace(/（.*?）|\(.*?\)/g, "")
        .split(/[、,]/);

      for (const ja of list) {
        const key = ja.trim();
        if (key) jaToSimne[key] = simne;
      }
    }
  }

  dictLoaded = true;
  document.getElementById("status").textContent =
    `辞書ロード完了：${Object.keys(jaToSimne).length}語`;
}

/* スペースなし日本語を分割 */
function tokenizeJapanese(text) {
  const tokens = [];
  let i = 0;

  const dictWords = Object.keys(jaToSimne)
    .sort((a, b) => b.length - a.length);

  while (i < text.length) {
    let matched = false;

    for (const w of dictWords) {
      if (text.startsWith(w, i)) {
        tokens.push(w);
        i += w.length;
        matched = true;
        break;
      }
    }

    if (!matched) {
      tokens.push(text[i]);
      i++;
    }
  }
  return tokens;
}

/* 翻訳 */
async function translate() {
  await loadDictionary();

  const input = document.getElementById("input").value.trim();
  if (!input) return;

  const words = input.includes(" ")
    ? input.split(/\s+/)
    : tokenizeJapanese(input);

  let S=[], O=[], V=[], mode="S";

  for (const w of words) {
    if (["は","が"].includes(w)) { mode="O"; continue; }
    if (w === "を") { mode="V"; continue; }

    if (mode==="S") S.push(w);
    else if (mode==="O") O.push(w);
    else V.push(w);
  }

  const ordered = [...S, ...V, ...O];

  const result = ordered.map(w => {
    return jaToSimne[w]
      ? jaToSimne[w]
      : `<span class="unknown">${w}</span>`;
  }).join(" ");

  document.getElementById("output").innerHTML = result;
}

/* 辞書検索 */
document.getElementById("dictSearch").addEventListener("input", async e => {
  await loadDictionary();
  const q = e.target.value.trim();
  const out = document.getElementById("dictSearchResult");

  if (!q) {
    out.textContent = "";
    return;
  }

  out.textContent = jaToSimne[q]
    ? `${q} → ${jaToSimne[q]}`
    : "未登録";
});
</script>

</body>
</html>

