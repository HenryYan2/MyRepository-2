<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>SKILL 字串轉碼工具（Big5 / GBK ↔ 八進位跳脫）</title>
<style>
  :root {
    --bg: #f4f6fa;
    --card: #ffffff;
    --ink: #1f2733;
    --muted: #6b7686;
    --line: #e2e8f0;
    --accent: #2563eb;
    --accent-soft: #eff4ff;
    --ok: #16a34a;
    --warn: #d97706;
    --mono: "Cascadia Code", "Consolas", "SFMono-Regular", Menlo, monospace;
  }
  * { box-sizing: border-box; }
  body {
    margin: 0;
    min-height: 100vh;
    background:
      radial-gradient(1200px 500px at 10% -10%, #e7efff 0%, transparent 60%),
      radial-gradient(1000px 500px at 110% 10%, #eafaf1 0%, transparent 55%),
      var(--bg);
    color: var(--ink);
    font-family: "Segoe UI", "Microsoft JhengHei", "PingFang TC", system-ui, sans-serif;
    display: flex;
    justify-content: center;
    padding: 40px 16px 60px;
  }
  .card {
    width: 100%;
    max-width: 760px;
    background: var(--card);
    border: 1px solid var(--line);
    border-radius: 18px;
    box-shadow: 0 18px 50px rgba(31, 39, 51, .10);
    padding: 28px 30px 26px;
  }
  h1 {
    margin: 0 0 4px;
    font-size: 21px;
    letter-spacing: .3px;
  }
  .subtitle {
    margin: 0 0 20px;
    color: var(--muted);
    font-size: 13px;
  }
  .row {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 10px;
    margin-bottom: 14px;
  }
  .seg {
    display: inline-flex;
    border: 1px solid var(--line);
    border-radius: 10px;
    overflow: hidden;
    background: #f9fafc;
  }
  .seg button {
    border: 0;
    background: transparent;
    padding: 8px 16px;
    font-size: 13.5px;
    cursor: pointer;
    color: var(--muted);
    font-family: inherit;
    transition: all .15s ease;
  }
  .seg button + button { border-left: 1px solid var(--line); }
  .seg button.active {
    background: var(--accent-soft);
    color: var(--accent);
    font-weight: 600;
  }
  .lbl { font-size: 13px; color: var(--muted); margin-right: 2px; }
  .dir-switch {
    margin-left: auto;
    display: inline-flex;
    align-items: center;
    gap: 8px;
  }
  .dir-switch .track {
    width: 46px; height: 26px;
    border-radius: 999px;
    background: #d6dde8;
    position: relative;
    cursor: pointer;
    transition: background .2s ease;
    flex: none;
  }
  .dir-switch .track::after {
    content: "";
    position: absolute;
    top: 3px; left: 3px;
    width: 20px; height: 20px;
    border-radius: 50%;
    background: #fff;
    box-shadow: 0 1px 3px rgba(0,0,0,.25);
    transition: left .2s ease;
  }
  .dir-switch .track.on {
    background: var(--accent);
  }
  .dir-switch .track.on::after { left: 23px; }
  .dir-kind { font-size: 13px; color: var(--muted); min-width: 150px; }
  .dir-kind b { color: var(--ink); }

  .box {
    width: 100%;
    border: 1.5px solid var(--line);
    border-radius: 12px;
    padding: 12px 14px;
    font-family: var(--mono);
    font-size: 15px;
    color: var(--ink);
    resize: vertical;
    min-height: 86px;
    line-height: 1.6;
    background: #fbfcfe;
    outline: none;
    transition: border-color .15s ease, box-shadow .15s ease;
  }
  .box:focus {
    border-color: var(--accent);
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12);
  }
  .box[readonly] { background: #f6f8fb; cursor: text; }
  .box-label {
    display: block;
    font-size: 12.5px;
    color: var(--muted);
    margin: 14px 0 6px;
  }
  .caps { text-transform: uppercase; letter-spacing: .4px; }

  .flow { text-align: center; color: var(--accent); font-size: 16px; margin: 2px 0; }

  .actions { display: flex; align-items: center; gap: 10px; margin-top: 14px; }
  .btn {
    border: 1px solid var(--line);
    background: #fff;
    color: var(--ink);
    border-radius: 9px;
    padding: 8px 16px;
    font-size: 13.5px;
    cursor: pointer;
    font-family: inherit;
    transition: all .15s ease;
  }
  .btn:hover { border-color: var(--accent); color: var(--accent); }
  .btn.primary {
    background: var(--accent);
    border-color: var(--accent);
    color: #fff;
  }
  .btn.primary:hover { background: #1d4ed8; }
  .btn.copied { background: var(--ok); border-color: var(--ok); color: #fff; }

  .samples { display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
  .chip {
    font-size: 12px;
    color: var(--muted);
    border: 1px dashed #c6d0de;
    border-radius: 999px;
    padding: 3px 10px;
    cursor: pointer;
    background: #fbfcfe;
    transition: all .15s ease;
  }
  .chip:hover { color: var(--accent); border-color: var(--accent); }

  .status {
    margin-top: 16px;
    font-size: 12.5px;
    color: var(--muted);
    border-top: 1px dashed var(--line);
    padding-top: 12px;
    min-height: 20px;
  }
  .status .tag {
    display: inline-block;
    background: var(--accent-soft);
    color: var(--accent);
    border-radius: 6px;
    padding: 1px 8px;
    font-weight: 600;
    margin-right: 8px;
  }
  .warn-msg { color: var(--warn); margin-top: 6px; }

  details.detail {
    margin-top: 14px;
    border: 1px solid var(--line);
    border-radius: 10px;
    padding: 10px 14px;
    background: #fbfcfe;
  }
  details.detail summary {
    cursor: pointer;
    font-size: 13px;
    color: var(--muted);
    user-select: none;
  }
  table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 8px;
    font-family: var(--mono);
    font-size: 13px;
  }
  th, td {
    text-align: left;
    padding: 5px 8px;
    border-bottom: 1px solid var(--line);
  }
  th { color: var(--muted); font-weight: 600; }
  td.ch { font-family: "Microsoft JhengHei", "PingFang TC", sans-serif; font-size: 15px; }
  .hexcol { color: var(--accent); }
</style>
</head>
<body>
<div class="card">
  <h1>SKILL 字串轉碼工具</h1>
  <p class="subtitle">繁體中文 Big5 ⬌ 簡體中文 GBK ⬌ SKILL 八進位跳脫（<span class="caps">\ooo</span>）即時互轉</p>

  <div class="row">
    <span class="lbl">編碼：</span>
    <div class="seg" id="encSeg">
      <button data-enc="big5" class="active">Big5 繁中</button>
      <button data-enc="gbk">GBK 簡中</button>
    </div>
    <div class="dir-switch" id="dirSwitch">
      <div class="track" id="dirTrack"></div>
      <span class="dir-kind" id="dirKind"><b>中文 → 跳脫</b></span>
    </div>
  </div>

  <label class="box-label caps" for="inBox">輸入（原始文字 或 跳脫字串）</label>
  <textarea id="inBox" class="box" rows="3" spellcheck="false" placeholder="例如：博智 或 輸入下面跳脫字串試試看&#10;\263\325\264\274"></textarea>

  <div class="flow">▼ 自動轉換 ▼</div>

  <label class="box-label caps" for="outBox">輸出（結果）</label>
  <textarea id="outBox" class="box" rows="3" readonly spellcheck="false" placeholder="轉換結果會自動顯示在這裡"></textarea>

  <div class="actions">
    <button class="btn primary" id="copyBtn">複製結果</button>
    <span class="samples" id="samples">
      <span class="lbl">範例：</span>
      <span class="chip" data-s="博智">博智</span>
      <span class="chip" data-s="耀新">耀新</span>
      <span class="chip" data-s="蘇州金像">蘇州金像</span>
      <span class="chip" data-s="中文測試：板號 V1.2">中文測試</span>
    </span>
  </div>

  <div class="status" id="status">等待輸入…</div>

  <details class="detail">
    <summary>字元對照（位元組 HEX / 八進位）</summary>
    <div id="detailBox"></div>
  </details>
</div>

<script>
(function () {
  'use strict';

  const $ = (id) => document.getElementById(id);
  const inBox = $('inBox'), outBox = $('outBox');
  const statusEl = $('status'), detailBox = $('detailBox');
  const copyBtn = $('copyBtn');
  const dirTrack = $('dirTrack'), dirKind = $('dirKind');

  let currentEnc = 'big5';   // 'big5' | 'gbk'
  let direction = 'enc';     // 'enc' (中文→跳脫) | 'dec' (跳脫→中文)

  const encLabel = { big5: 'Big5（繁體中文）', gbk: 'GBK（簡體中文）' };

  // ---------- 編碼表（TextDecoder 反查建立 Char → [b1,b2]） ----------
  const encoders = {};
  function getMap(enc) {
    if (encoders[enc]) return encoders[enc];
    const map = new Map();
    const dec = new TextDecoder(enc);
    for (let b1 = 0x81; b1 <= 0xFE; b1++) {
      for (let b2 = 0x40; b2 <= 0xFE; b2++) {
        if (b2 === 0x7F) continue;
        const s = dec.decode(new Uint8Array([b1, b2]));
        if (s.length === 1 && s !== '\uFFFD' && !map.has(s)) {
          map.set(s, [b1, b2]);
        }
      }
    }
    encoders[enc] = map;
    return map;
  }

  const toHex = (b) => b.toString(16).toUpperCase().padStart(2, '0');
  const toOct  = (b) => '\\' + b.toString(8).padStart(3, '0');

  // ---------- 中文 → 八進位跳脫 ----------
  function encodeString(str) {
    const map = getMap(currentEnc);
    const rows = [];
    const missing = [];
    for (const ch of str) {
      const cp = ch.codePointAt(0);
      if (cp < 0x80) {
        rows.push({ ch, bytes: [cp], esc: toOct(cp), note: 'ASCII' });
        continue;
      }
      const b = map.get(ch);
      if (!b) { missing.push(ch); rows.push({ ch, bytes: null, esc: null, note: '無法編碼' }); continue; }
      rows.push({ ch, bytes: b, esc: toOct(b[0]) + toOct(b[1]) });
    }
    return rows;
  }

  // ---------- 八進位跳脫 → 中文 ----------
  function parseEscapes(str) {
    let s = str.trim();
    const q1 = s.startsWith('"') && s.endsWith('"');
    const q2 = s.startsWith("'") && s.endsWith("'");
    if ((q1 || q2) && s.length >= 2) s = s.slice(1, -1);

    const bytes = [];
    const re = /\\([0-7]{1,3})/g;
    let m, pos = 0;
    while ((m = re.exec(s)) !== null) {
      for (let i = pos; i < m.index; i++) bytes.push(s.charCodeAt(i) & 0xFF);
      bytes.push(parseInt(m[1], 8) & 0xFF);
      pos = m.index + m[0].length;
    }
    for (let i = pos; i < s.length; i++) bytes.push(s.charCodeAt(i) & 0xFF);
    return new Uint8Array(bytes);
  }

  // ---------- 畫面渲染 ----------
  function refresh() {
    const raw = inBox.value;
    if (!raw.trim()) {
      outBox.value = '';
      statusEl.innerHTML = '等待輸入…';
      detailBox.innerHTML = '';
      return;
    }

    if (direction === 'enc') {
      const rows = encodeString(raw);
      const quote = '"' + rows.map(r => r.esc || '').join('') + '"';
      outBox.value = quote;

      const ok = rows.filter(r => r.bytes).length;
      const bad = rows.filter(r => !r.bytes);
      const totalBytes = rows.reduce((n, r) => n + (r.bytes ? r.bytes.length : 0), 0);
      const hex = rows.filter(r => r.bytes).map(r => r.bytes.map(toHex).join('')).join(' ');

      let html = '';
      let msg;
      if (bad.length) {
        msg = `<span class="warn-msg">⚠ ${bad.map(r => r.ch).join('、')} 無法用 ${encLabel[currentEnc]} 編碼（常用字應可，試試切換另一種編碼）</span>`;
      } else {
        msg = '✔ 全部字元均可編碼';
      }
      statusEl.innerHTML =
        `<span class="tag">${currentEnc.toUpperCase()} · ${encLabel[currentEnc]}</span>` +
        `字元 ${raw.length} 個 · 位元組 ${totalBytes} B<br>` +
        `HEX：<span class="hexcol">${hex || '—'}</span><br>${msg}`;

      if (rows.some(r => r.bytes)) {
        let tr = '<table><tr><th>字元</th><th>編碼</th><th>HEX</th><th>八進位跳脫</th><th></th></tr>';
        for (const r of rows) {
          if (!r.bytes) { tr += `<tr><td class="ch">${r.ch}</td><td colspan="4">⚠ 無法編碼（此編碼無對應字元）</td></tr>`; continue; }
          tr += `<tr><td class="ch">${r.ch}</td><td>${r.note || (currentEnc.toUpperCase())}</td>` +
                `<td class="hexcol">${r.bytes.map(toHex).join(' ')}</td>` +
                `<td>${r.esc}</td><td>${r.bytes.length === 1 ? 'ASCII' : ''}</td></tr>`;
        }
        tr += '</table>';
        detailBox.innerHTML = tr;
      } else {
        detailBox.innerHTML = '';
      }
    } else {
      const bytes = parseEscapes(raw);
      const dec = new TextDecoder(currentEnc, { fatal: false });
      const out = dec.decode(bytes);
      outBox.value = out;

      const hex = Array.from(bytes).map(toHex).join(' ');
      statusEl.innerHTML =
        `<span class="tag">${currentEnc.toUpperCase()} · ${encLabel[currentEnc]}</span>` +
        `解析位元組 ${bytes.length} B &nbsp; HEX：<span class="hexcol">${hex || '—'}</span><br>` +
        (bytes.length % 2 !== 0 ? '<span class="warn-msg">⚠ 位元組數為奇數，可能有多餘/不完整位元組</span>' : '') +
        (out.includes('\uFFFD') ? '<span class="warn-msg">⚠ 含替換字元 �，可能編碼不符或輸入有誤</span>' : '');

      if (bytes.length) {
        let tr = '<table><tr><th>#</th><th>HEX</th><th>八進位</th></tr>';
        const arr = Array.from(bytes);
        for (let i = 0; i < arr.length; i += 2) {
          const pair = [arr[i], arr[i + 1] ?? 0];
          tr += `<tr><td>${i / 2 + 1}</td><td class="hexcol">${pair.map(toHex).join(' ')}</td>` +
                `<td>${toOct(pair[0])}${i + 1 < arr.length ? toOct(pair[1]) : ''}</td></tr>`;
        }
        tr += '</table>';
        detailBox.innerHTML = tr;
      } else {
        detailBox.innerHTML = '';
      }
    }
  }

  // ---------- 事件 ----------
  inBox.addEventListener('input', refresh);

  document.querySelectorAll('#encSeg button').forEach(btn => {
    btn.addEventListener('click', () => {
      document.querySelectorAll('#encSeg button').forEach(b => b.classList.remove('active'));
      btn.classList.add('active');
      currentEnc = btn.dataset.enc;
      refresh();
    });
  });

  dirTrack.addEventListener('click', () => {
    direction = direction === 'enc' ? 'dec' : 'enc';
    dirTrack.classList.toggle('on', direction === 'dec');
    dirKind.innerHTML = direction === 'enc' ? '<b>中文 → 跳脫</b>' : '<b>跳脫 → 中文</b>';
    inBox.placeholder = direction === 'enc'
      ? '例如：博智'
      : '例如：\\263\\325\\264\\274';
    refresh();
  });

  outBox.addEventListener('click', () => {
    outBox.select();
  });

  copyBtn.addEventListener('click', async () => {
    const v = outBox.value;
    if (!v) return;
    try {
      await navigator.clipboard.writeText(v);
    } catch (e) {
      outBox.select();
      document.execCommand('copy');
    }
    copyBtn.textContent = '已複製 ✓';
    copyBtn.classList.add('copied');
    setTimeout(() => {
      copyBtn.textContent = '複製結果';
      copyBtn.classList.remove('copied');
    }, 1400);
  });

  document.querySelectorAll('#samples .chip').forEach(chip => {
    chip.addEventListener('click', () => {
      if (direction === 'dec') { dirTrack.click(); }
      inBox.value = chip.dataset.s;
      refresh();
    });
  });

  refresh();
})();
</script>
</body>
</html>
