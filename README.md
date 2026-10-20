Agentic OS - Full Stack Multi-Pattern Orche:

Turn Claude Code from a CLI into an autonomous workflow platform that routes ALL work through optimal patterns, runs on autopilot, and learns from every execution.

What You Get

6 Orchestration Patterns

Pipeline - Sequential dependencies (analyze → transform → validate)

Fan-out - Parallel experts (technical + business + creative → consolidate)

• Expert Pool - Route by task type (classification → specialist → execute)

Producer-Reviewer - Draft → QA gate (95/100) → Revise

.

Supervisor - Plan → Delegate → Validate → Consolidate

Hierarchical - Multi-level authority chain (CEO → VP → Manager → IC)

13 Specialized Agents

Customer Service: customer-advocate, qa-reviewer Content: content- strategist, content-creator, content-editor Data: data-analyst, data-engineer Research: researcher, fact-checker Operations: project-manager, compliance-officer Tech: code-reviewer System: orchestrator

✓ 11 Composable Skills

Customer Service: analyze-ticket, generate-response Content: outline- content, optimize-seo, adapt-formats Data: clean-data, generate-insights Research: research-topic, fact-check Operations: plan-project, compliance- check

✓ 4 Interfaces

1. CLI - Terminal commands for running tasks

2. API - RESTful endpoints for automation

3. Dashboard - React Ul for one-click execution

4. Memory System - Self-improving feedback loops# AI-llm
<!doctype html><title>AI Copilot</title><p>Start Termux: bash ~/copilot/start.sh</p>{
  "appId": "com.copilot.pro",
  "appName": "AI Copilot",
  "webDir": "www",
  "server": { "url": "http://localhost:8080", "cleartext": true }
}#!/data/data/com.termux/files/usr/bin/bash
set -e
pkg update -y && pkg upgrade -y
pkg install -y nodejs
mkdir -p ~/copilot/public && cd ~/copilot

# ---------- .env (created once) ----------
if [ ! -f .env ]; then
TOKEN=$(node -e "console.log(require('crypto').randomBytes(16).toString('hex'))")
cat > .env <<EOF
ANTHROPIC_API_KEY=PASTE_YOUR_KEY_HERE
BRIDGE_TOKEN=$TOKEN
MODEL=claude-sonnet-5-5
EOF
fi

# ---------- start.sh ----------
cat > start.sh <<'EOF_START'
#!/data/data/com.termux/files/usr/bin/bash
cd ~/copilot
termux-wake-lock 2>/dev/null || true
node server.js
EOF_START
chmod +x start.sh

# ---------- server.js (bridge + agent pipeline) ----------
cat > server.js <<'EOF_SERVER'
const http = require('http');
const fs = require('fs');
const path = require('path');
const { execFile } = require('child_process');

try {
  fs.readFileSync(path.join(__dirname, '.env'), 'utf8').split('\n').forEach(l => {
    const m = l.match(/^\s*([A-Z_]+)\s*=\s*(.*?)\s*$/);
    if (m) process.env[m[1]] = m[2].replace(/^["']|["']$/g, '');
  });
} catch (e) {}

const PORT = 8080;
const KEY = process.env.ANTHROPIC_API_KEY;
const TOKEN = process.env.BRIDGE_TOKEN;
const MODEL = process.env.MODEL || 'claude-sonnet-5-5';
const ALLOWED = new Set(['ls', 'pwd', 'date', 'whoami', 'uname', 'df', 'uptime']);
const PUB = path.join(__dirname, 'public');
const MIME = { '.html': 'text/html', '.js': 'text/javascript', '.json': 'application/json',
  '.svg': 'image/svg+xml', '.webmanifest': 'application/manifest+json' };

async function claude(system, messages, max) {
  if (!KEY || KEY.includes('PASTE')) throw new Error('Set ANTHROPIC_API_KEY in ~/copilot/.env');
  const r = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: { 'content-type': 'application/json', 'x-api-key': KEY, 'anthropic-version': '2023-06-01' },
    body: JSON.stringify({ model: MODEL, max_tokens: max, system, messages })
  });
  const j = await r.json();
  if (!r.ok) throw new Error((j.error && j.error.message) || 'HTTP ' + r.status);
  return j.content.map(b => b.text || '').join('');
}

const PHONE = ' Keep the answer phone-friendly: concise, no filler.';
const SPECIALISTS = {
  code: 'You are the Code Agent. Give correct, runnable code with a short explanation.' + PHONE,
  analyze: 'You are the Analyst Agent. Find patterns, quantify, state assumptions and limits.' + PHONE,
  write: 'You are the Writing Agent. Produce clear, well-structured content in the requested tone.' + PHONE,
  research: 'You are the Research Agent. Synthesize what you know, separate facts from uncertainty, say what to verify.' + PHONE,
  solve: 'You are the Problem-Solving Agent. Diagnose, then give numbered steps.' + PHONE
};

async function pipeline(history, text, send) {
  send({ agent: 'Orchestrator', status: 'working' });
  let route = (await claude(
    'Classify the request into exactly one word: code, analyze, write, research, or solve. Reply with the word only.',
    [{ role: 'user', content: text }], 10)).trim().toLowerCase();
  if (!SPECIALISTS[route]) route = 'solve';
  send({ agent: 'Orchestrator', status: 'done', note: 'routed to ' + route });

  send({ agent: route + ' agent', status: 'working' });
  const draft = await claude(SPECIALISTS[route], history.concat([{ role: 'user', content: text }]), 1800);
  send({ agent: route + ' agent', status: 'done' });

  send({ agent: 'Reviewer', status: 'working' });
  const final = await claude(
    'You are the Reviewer Agent. Check the draft for errors, gaps and unclear parts. Return the corrected final answer only, with no commentary about the review.',
    [{ role: 'user', content: 'Request:\n' + text + '\n\nDraft:\n' + draft }], 2000);
  send({ agent: 'Reviewer', status: 'done' });
  return final;
}

function readBody(req) {
  return new Promise(resolve => {
    let b = '';
    req.on('data', c => { b += c; if (b.length > 1e6) req.destroy(); });
    req.on('end', () => { try { resolve(JSON.parse(b || '{}')); } catch (e) { resolve({}); } });
  });
}
function json(res, code, obj) {
  res.writeHead(code, { 'content-type': 'application/json' });
  res.end(JSON.stringify(obj));
}

async function api(req, res, url) {
  if (!TOKEN || req.headers['x-token'] !== TOKEN) return json(res, 401, { error: 'unauthorized' });
  if (url.pathname === '/api/ping') return json(res, 200, { ok: true });
  const b = await readBody(req);

  if (url.pathname === '/api/chat') {
    res.writeHead(200, { 'content-type': 'application/x-ndjson', 'cache-control': 'no-cache' });
    const send = o => res.write(JSON.stringify(o) + '\n');
    try { send({ final: await pipeline(b.history || [], String(b.text || ''), send) }); }
    catch (e) { send({ error: e.message }); }
    return res.end();
  }

  if (url.pathname === '/api/exec') {
    const parts = String(b.command || '').trim().split(/\s+/);
    if (!ALLOWED.has(parts[0]))
      return json(res, 403, { output: 'Not allowed. Allowed: ' + [...ALLOWED].join(', ') });
    return execFile(parts[0], parts.slice(1), { timeout: 10000 }, (err, out, errout) =>
      json(res, 200, { output: out || errout || String(err || '') }));
  }
  json(res, 404, { error: 'not found' });
}

http.createServer((req, res) => {
  const url = new URL(req.url, 'http://x');
  if (url.pathname.startsWith('/api/')) return api(req, res, url);
  const p = url.pathname === '/' ? '/index.html' : url.pathname;
  const f = path.normalize(path.join(PUB, p));
  if (!f.startsWith(PUB)) { res.writeHead(403); return res.end(); }
  fs.readFile(f, (e, d) => {
    if (e) { res.writeHead(404); return res.end('Not found'); }
    res.writeHead(200, { 'content-type': MIME[path.extname(f)] || 'application/octet-stream' });
    res.end(d);
  });
}).listen(PORT, '127.0.0.1', () => console.log('AI Copilot running: http://localhost:' + PORT));
EOF_SERVER

# ---------- public/index.html ----------
cat > public/index.html <<'EOF_HTML'
<!doctype html>
<html lang="en"><head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<meta name="theme-color" content="#5b21b6">
<link rel="manifest" href="manifest.webmanifest">
<title>AI Copilot</title>
<style>
:root{--bg:#0b0b14;--card:#1a1730;--p:#7c3aed;--t:#eee}
*{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
body{margin:0;background:var(--bg);color:var(--t);font:15px system-ui,sans-serif;height:100dvh;display:flex;flex-direction:column}
header{padding:12px 16px;background:#5b21b6;font-weight:700;display:flex;justify-content:space-between}
#clr{font-weight:400;opacity:.8}
#chips{display:flex;gap:8px;padding:8px;overflow-x:auto;flex-shrink:0}
#chips button{background:var(--card);color:var(--t);border:1px solid #3b2f78;border-radius:16px;padding:8px 14px;white-space:nowrap}
#log{flex:1;overflow-y:auto;padding:12px}
.m{max-width:90%;margin:6px 0;padding:10px 12px;border-radius:14px;white-space:pre-wrap;word-wrap:break-word}
.u{background:var(--p);margin-left:auto}
.a{background:var(--card)}
.tr{font-size:12px;opacity:.7}
#f{display:flex;gap:8px;padding:8px;padding-bottom:calc(8px + env(safe-area-inset-bottom))}
input{flex:1;min-width:0;background:var(--card);color:var(--t);border:1px solid #3b2f78;border-radius:12px;padding:12px;font-size:16px}
#f button{border:0;border-radius:12px;background:var(--p);color:#fff;padding:0 14px;font-size:16px}
</style></head><body>
<header><span>🤖 AI Copilot</span><span id="clr">Clear</span></header>
<div id="chips">
<button data-p="Write code that ">Code</button>
<button data-p="Analyze this: ">Analyze</button>
<button data-p="Write: ">Write</button>
<button data-p="Research: ">Research</button>
<button data-p="Help me solve: ">Solve</button>
<button data-p="/run ">Terminal</button>
</div>
<div id="log"></div>
<form id="f">
<button type="button" id="mic">🎤</button>
<input id="q" placeholder="Ask anything…" autocomplete="off">
<button>Send</button>
</form>
<script src="firebase-config.js"></script>
<script src="firebase-sync.js"></script>
<script>
const $ = s => document.querySelector(s), log = $('#log');
let token = localStorage.token || prompt('Bridge token (BRIDGE_TOKEN from ~/copilot/.env)') || '';
localStorage.token = token;
let hist = JSON.parse(localStorage.hist || '[]');
const save = () => localStorage.hist = JSON.stringify(hist.slice(-30));
const add = (cls, t) => {
  const d = document.createElement('div');
  d.className = 'm ' + cls; d.textContent = t;
  log.appendChild(d); log.scrollTop = log.scrollHeight; return d;
};
hist.forEach(m => add(m.role === 'user' ? 'u' : 'a', m.content));
$('#clr').onclick = () => { hist = []; save(); log.innerHTML = ''; };
document.querySelectorAll('#chips button').forEach(b =>
  b.onclick = () => { $('#q').value = b.dataset.p; $('#q').focus(); });

const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
if (SR) {
  const rec = new SR();
  rec.onresult = e => $('#q').value = e.results[0][0].transcript;
  $('#mic').onclick = () => rec.start();
} else { $('#mic').hidden = true; }

$('#f').onsubmit = async e => {
  e.preventDefault();
  const text = $('#q').value.trim();
  if (!text) return;
  $('#q').value = '';
  add('u', text);
  const H = { 'x-token': token, 'content-type': 'application/json' };

  if (text.startsWith('/run ')) {
    try {
      const r = await fetch('/api/exec', { method: 'POST', headers: H, body: JSON.stringify({ command: text.slice(5) }) });
      const j = await r.json();
      add('a', j.output || j.error || 'No output');
    } catch (err) { add('a', 'Error: ' + err.message); }
    return;
  }

  const tr = add('a tr', 'Starting…');
  try {
    const r = await fetch('/api/chat', { method: 'POST', headers: H, body: JSON.stringify({ text, history: hist.slice(-10) }) });
    if (r.status === 401) { localStorage.removeItem('token'); tr.textContent = 'Wrong token. Reload and re-enter it.'; return; }
    const rd = r.body.getReader(), dec = new TextDecoder();
    let buf = '';
    for (;;) {
      const { done, value } = await rd.read();
      if (done) break;
      buf += dec.decode(value, { stream: true });
      let i;
      while ((i = buf.indexOf('\n')) >= 0) {
        const line = buf.slice(0, i); buf = buf.slice(i + 1);
        if (!line) continue;
        const o = JSON.parse(line);
        if (o.agent) tr.textContent = '⚙ ' + o.agent + (o.status === 'done' ? ' ✓' : '…') + (o.note ? ' (' + o.note + ')' : '');
        if (o.final) {
          tr.className = 'm a'; tr.textContent = o.final;
          hist.push({ role: 'user', content: text }, { role: 'assistant', content: o.final });
          save();
          if (window.cloudSave) { cloudSave('user', text); cloudSave('assistant', o.final); }
        }
        if (o.error) tr.textContent = 'Error: ' + o.error;
      }
    }
  } catch (err) { tr.textContent = 'Error: ' + err.message + ' (is start.sh running in Termux?)'; }
};
</script></body></html>
EOF_HTML

# ---------- manifest + icon ----------
cat > public/manifest.webmanifest <<'EOF_MANIFEST'
{"name":"AI Copilot","short_name":"Copilot","start_url":"/","display":"standalone","background_color":"#0b0b14","theme_color":"#5b21b6","icons":[{"src":"icon.svg","sizes":"any","type":"image/svg+xml","purpose":"any"}]}
EOF_MANIFEST

cat > public/icon.svg <<'EOF_ICON'
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512"><rect width="512" height="512" rx="110" fill="#5b21b6"/><circle cx="256" cy="256" r="120" fill="none" stroke="#fff" stroke-width="28"/><circle cx="256" cy="256" r="36" fill="#fff"/></svg>
EOF_ICON

# ---------- Firebase (inactive until you add config) ----------
cat > public/firebase-config.js <<'EOF_FBC'
// Replace null with your config object to enable cloud history:
// window.FIREBASE_CONFIG = { apiKey:"...", authDomain:"...", databaseURL:"...", projectId:"...", appId:"..." };
window.FIREBASE_CONFIG = null;
EOF_FBC

cat > public/firebase-sync.js <<'EOF_FBS'
(function () {
  if (!window.FIREBASE_CONFIG) return;
  const base = 'https://www.gstatic.com/firebasejs/10.12.2/';
  const load = s => new Promise(r => { const e = document.createElement('script'); e.src = s; e.onload = r; document.head.appendChild(e); });
  (async () => {
    await load(base + 'firebase-app-compat.js');
    await load(base + 'firebase-auth-compat.js');
    await load(base + 'firebase-database-compat.js');
    firebase.initializeApp(window.FIREBASE_CONFIG);
    const cred = await firebase.auth().signInAnonymously();
    const uid = cred.user.uid;
    window.cloudSave = (role, text) =>
      firebase.database().ref('users/' + uid + '/messages').push({ role, text, t: Date.now() });
  })().catch(e => console.warn('Firebase:', e));
})();
EOF_FBS

echo ""
echo "DONE. Your bridge token:"
grep BRIDGE_TOKEN .env
echo "Next: nano ~/copilot/.env  (paste your ANTHROPIC_API_KEY), then: bash ~/copilot/start.sh"nano ~/copilot/.env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