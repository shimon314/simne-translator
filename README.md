<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>日本語 → シムネ語 翻訳機</title>

<style>
body {
  background:#0e1117;
  color:#e6edf3;
  font-family:-apple-system, BlinkMacSystemFont, sans-serif;
  padding:16px;
}
h1 { font-size:22px; margin-bottom:8px; }
textarea, input {
  width:100%;
  background:#161b22;
  color:#e6edf3;
  border:1px solid #30363d;
  border-radius:6px;
  padding:10px;
  margin-bottom:10px;
}
button {
  width:100%;
  padding:12px;
  font-size:16px;
  background:#238636;
  color:white;
  border:none;
  border-radius:6px;
}
button:active { opacity:0.8; }
#status {
  font-size:13px;
  margin-bottom:10px;
}
.ok { color:#3fb950; }
.err { color:#f85149; }
#output span.unknown {
  color:#f85149;
  text-decoration:underline;
}
.card {
  background:#161b22;
  border:1px solid #30363d;
  border-radius:6px;
  padding:10px;
  margin-bottom:12px;
}
</style>
</head>

<body>

<h1>日本語 → シムネ語 翻訳機</h1>
<div id="status">辞書ロード中…</div>

<div class="card">
<textarea id="input" placeholder="例：私は世界を見る"></textarea>
<button id="translateBtn">翻訳</button>
</div>

<div class="card">
<h3>翻訳結果</h3>
<div id="output"></div>
</div>

<div class="card">
<h3>辞書検索（日本語）</h3>
<input id="dictSearch" placeholder="例：私 / 世界 / 見る">
<div id="dictResult"></div>
</div>

<script>
const DICT_FILE = "シムネ語 詩日辞書.json";

let jaToSimne = {};
let dictReady = false;

/* ========= 辞書ロード ========= */
async function loadDictionary() {
  try {
    const res = await fetch(DICT_FILE, { cache: "no-store" });
    if (!res.ok) throw new Error("fetch失敗");

    const data = await res.json();
    if (!data.words) throw new Error("JSON構造不正");

    for (const w of data.words) {
      const simne = w.entry?.form;
      if (!simne) continue;

      for (const tr of w.translations ?? []) {
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

    dictReady = true;
    document.getElementById("status").innerHTML =
      `<span class="ok">辞書ロード完了：${Object.keys(jaToSimne).length}語</span>`;
  } catch (e) {
    document.getElementById("status").innerHTML =
      `<span class="err">辞書ロード失敗：${e.message}</span>`;
  }
}

/* ========= 日本語分割（スペースなし対応） ========= */
function tokenize(text) {
  const tokens = [];
  let i = 0;
  const dictWords = Object.keys(jaToSimne).sort((a,b)=>b.length-a.length);

  while (i < text.length) {
    let hit = false;
    for (const w of dictWords) {
      if (text.startsWith(w, i)) {
        tokens.push(w);
        i += w.length;
        hit = true;
        break;
      }
    }
    if (!hit) {
      tokens.push(text[i]);
      i++;
    }
  }
  return tokens;
}

/* ========= 翻訳 ========= */
function translateText() {
  if (!dictReady) return;

  const input = document.getElementById("input").value.trim();
  if (!input) return;

  const words = input.includes(" ")
    ? input.split(/\s+/)
    : tokenize(input);

  let S=[], O=[], V=[], mode="S";

  for (const w of words) {
    if (["は","が"].includes(w)) { mode="O"; continue; }
    if (w==="を") { mode="V"; continue; }

    if (mode==="S") S.push(w);
    else if (mode==="O") O.push(w);
    else V.push(w);
  }

  const ordered = [...S, ...V, ...O];

  document.getElementById("output").innerHTML =
    ordered.map(w =>
      jaToSimne[w]
        ? jaToSimne[w]
        : `<span class="unknown">${w}</span>`
    ).join(" ");
}

/* ========= 辞書検索 ========= */
function searchDict(q) {
  if (!dictReady) return "";
  return jaToSimne[q] ?? "未登録";
}

/* ========= イベント ========= */
document.getElementById("translateBtn").onclick = translateText;

document.getElementById("dictSearch").addEventListener("input", e => {
  document.getElementById("dictResult").textContent =
    searchDict(e.target.value.trim());
});

/* ========= 初期化 ========= */
loadDictionary();
</script>

</body>
</html>
