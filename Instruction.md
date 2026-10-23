# Claude Code Chat UI for a Remote VM — Instructions

Goal: use Claude Code on your external VM through a friendly chat window (bubbles, Markdown, Stop and New-chat buttons) instead of the raw terminal. The window opens inside VS Code (Simple Browser panel) or in any normal browser.

This guide gives three options. Try them in order. Option 2 always works, because it needs only `claude` and `python3` on the VM.

| Option | What you get | Needs |
|---|---|---|
| 1. Official extension over Remote-SSH | Anthropic's own chat panel | Ability to install a VS Code extension on the VM |
| 2. Self-hosted Chat UI (main workaround) | Browser chat page wired to the `claude` CLI | `python3` (3.8+) and `claude` on the VM |
| 3. Terminal-in-editor | Claude in a full editor tab, one keypress | Nothing extra |

---

## 0. Prerequisites (check on the VM)

```bash
claude --version
python3 --version
claude -p "say hi"
```

- `claude -p "say hi"` must print a reply. If it asks you to log in, run `claude` once interactively and finish login first.
- Note the folder of the project you want Claude to work in, for example `/home/you/myproject`.

---

## Option 1 — Official extension (try first, 5 minutes)

Anthropic ships a VS Code extension, "Claude Code" (`anthropic.claude-code`). It gives a chat side panel. It often "does not show up" only because it was installed on the local side instead of the remote side.

1. On your local PC, install the VS Code extension **Remote - SSH** (`ms-vscode-remote.remote-ssh`).
2. `Ctrl+Shift+P` → **Remote-SSH: Connect to Host…** → `user@your-vm-ip`.
3. In the remote window, open the Extensions view, search **Claude Code**, and click **Install in SSH: your-vm**.
4. Open your project folder. Click the Claude icon in the sidebar or editor title bar.

If the VM has no Marketplace access (air-gapped or proxy):

1. On a machine with internet, open the Marketplace page for `anthropic.claude-code` and use **Download Extension** to get the `.vsix`.
2. Copy it to the VM (`scp claude-code.vsix user@vm:~/`).
3. On the VM: `code --install-extension ~/claude-code.vsix` (or Extensions view → `…` → **Install from VSIX…**).

If this works, stop here. If extensions are blocked by policy, use Option 2.

---

## Option 2 — Self-hosted Chat UI (main workaround)

### How it works

```
Browser / VS Code Simple Browser
        │  HTTP (localhost:8765, via SSH tunnel or VS Code port-forward)
        ▼
server.py on the VM  ──runs──►  claude -p "<msg>" --output-format stream-json --resume <session>
        ▲                                   │
        └──────── live JSON events ─────────┘
```

- Each message you send runs `claude -p` once in your project folder.
- The server remembers the `session_id`, so every next message uses `--resume` and the conversation keeps its full context.
- Replies stream live. Tool calls (file edits, bash commands) show as grey lines.
- The server binds only to `127.0.0.1`. Nobody else on the network can reach it.

### Step 1 — Create the folder

```bash
mkdir -p ~/claude-chat-ui
cd ~/claude-chat-ui
```

### Step 2 — Create `~/claude-chat-ui/server.py`

```python
#!/usr/bin/env python3
"""Minimal web chat front-end for the Claude Code CLI. Standard library only."""
import json
import os
import subprocess
import threading
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlparse

HOST = os.environ.get("CC_HOST", "127.0.0.1")
PORT = int(os.environ.get("CC_PORT", "8765"))
WORKDIR = os.path.expanduser(os.environ.get("CC_WORKDIR", os.getcwd()))
PERMISSION_MODE = os.environ.get("CC_PERMISSION_MODE", "acceptEdits")
CLAUDE_BIN = os.environ.get("CC_CLAUDE_BIN", "claude")
MODEL = os.environ.get("CC_MODEL", "")
HERE = os.path.dirname(os.path.abspath(__file__))

state = {"session_id": None, "proc": None}
run_lock = threading.Lock()


class Handler(BaseHTTPRequestHandler):
    def log_message(self, fmt, *args):
        pass

    def _send(self, code, body, ctype="application/json"):
        data = body if isinstance(body, bytes) else json.dumps(body).encode()
        self.send_response(code)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

    def do_GET(self):
        path = urlparse(self.path).path
        if path == "/":
            with open(os.path.join(HERE, "index.html"), "rb") as f:
                self._send(200, f.read(), "text/html; charset=utf-8")
        elif path == "/status":
            self._send(200, {"session_id": state["session_id"], "workdir": WORKDIR,
                             "busy": run_lock.locked()})
        else:
            self._send(404, {"error": "not found"})

    def do_POST(self):
        path = urlparse(self.path).path
        if path == "/reset":
            state["session_id"] = None
            self._send(200, {"ok": True})
        elif path == "/stop":
            proc = state["proc"]
            if proc and proc.poll() is None:
                proc.terminate()
            self._send(200, {"ok": True})
        elif path == "/chat":
            self._chat()
        else:
            self._send(404, {"error": "not found"})

    def _chat(self):
        length = int(self.headers.get("Content-Length", 0))
        msg = json.loads(self.rfile.read(length) or b"{}").get("message", "").strip()
        if not msg:
            return self._send(400, {"error": "empty message"})
        if not run_lock.acquire(blocking=False):
            return self._send(409, {"error": "Claude is still working on the previous message"})
        try:
            cmd = [CLAUDE_BIN, "-p", msg, "--output-format", "stream-json", "--verbose",
                   "--permission-mode", PERMISSION_MODE]
            if MODEL:
                cmd += ["--model", MODEL]
            if state["session_id"]:
                cmd += ["--resume", state["session_id"]]

            self.send_response(200)
            self.send_header("Content-Type", "text/event-stream")
            self.send_header("Cache-Control", "no-cache")
            self.end_headers()

            proc = subprocess.Popen(cmd, cwd=WORKDIR, stdout=subprocess.PIPE,
                                    stderr=subprocess.STDOUT, text=True, bufsize=1)
            state["proc"] = proc
            for line in proc.stdout:
                line = line.strip()
                if not line:
                    continue
                try:
                    event = json.loads(line)
                    if event.get("session_id"):
                        state["session_id"] = event["session_id"]
                except json.JSONDecodeError:
                    event = {"type": "raw", "text": line}
                self.wfile.write(f"data: {json.dumps(event)}\n\n".encode())
                self.wfile.flush()
            proc.wait()
            self.wfile.write(b'data: {"type": "done"}\n\n')
            self.wfile.flush()
        except (BrokenPipeError, ConnectionResetError):
            if state["proc"] and state["proc"].poll() is None:
                state["proc"].terminate()
        finally:
            state["proc"] = None
            run_lock.release()


if __name__ == "__main__":
    print(f"Claude Chat UI on http://{HOST}:{PORT}  (project: {WORKDIR})")
    ThreadingHTTPServer((HOST, PORT), Handler).serve_forever()
```

### Step 3 — Create `~/claude-chat-ui/index.html`

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Claude Chat</title>
<style>
  :root { --bg:#1e1e1e; --panel:#252526; --user:#0e639c; --bot:#2d2d30; --text:#e6e6e6; --muted:#9a9a9a; --accent:#d97757; }
  @media (prefers-color-scheme: light) {
    :root { --bg:#f6f6f6; --panel:#fff; --user:#0e639c; --bot:#fff; --text:#1e1e1e; --muted:#666; }
  }
  * { box-sizing: border-box; }
  body { margin:0; height:100vh; display:flex; flex-direction:column; background:var(--bg); color:var(--text);
         font:14px/1.5 system-ui, -apple-system, "Segoe UI", sans-serif; }
  header { padding:10px 16px; background:var(--panel); display:flex; gap:8px; align-items:center;
           border-bottom:1px solid #0003; }
  header b { color:var(--accent); }
  header .info { color:var(--muted); font-size:12px; flex:1; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
  button { background:#3a3d41; color:#fff; border:0; border-radius:6px; padding:6px 12px; cursor:pointer; }
  button.primary { background:var(--accent); }
  button:disabled { opacity:.5; cursor:default; }
  #log { flex:1; overflow-y:auto; padding:16px; display:flex; flex-direction:column; gap:10px; }
  .msg { max-width:min(900px, 92%); padding:10px 14px; border-radius:10px; white-space:pre-wrap; word-wrap:break-word; }
  .user { align-self:flex-end; background:var(--user); color:#fff; }
  .bot { align-self:flex-start; background:var(--bot); border:1px solid #0002; }
  .tool { align-self:flex-start; color:var(--muted); font:12px ui-monospace, Menlo, Consolas, monospace; }
  .meta { align-self:flex-start; color:var(--muted); font-size:11px; }
  pre { background:#0006; color:#e6e6e6; padding:10px; border-radius:6px; overflow-x:auto; white-space:pre; }
  code { font-family: ui-monospace, Menlo, Consolas, monospace; }
  form { display:flex; gap:8px; padding:12px 16px; background:var(--panel); border-top:1px solid #0003; }
  textarea { flex:1; resize:none; height:70px; padding:8px; border-radius:6px; border:1px solid #0004;
             background:var(--bg); color:var(--text); font:inherit; }
</style>
</head>
<body>
<header>
  <b>Claude Code</b>
  <span class="info" id="info">connecting…</span>
  <button id="stop" disabled>Stop</button>
  <button id="reset">New chat</button>
</header>
<div id="log"></div>
<form id="form">
  <textarea id="input" placeholder="Ask Claude… (Enter = send, Shift+Enter = new line)"></textarea>
  <button class="primary" id="send">Send</button>
</form>
<script>
const log = document.getElementById('log');
const input = document.getElementById('input');
const sendBtn = document.getElementById('send');
const stopBtn = document.getElementById('stop');

function esc(s) { return s.replace(/[&<>]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;'}[c])); }
function md(s) {
  // fenced code, inline code, bold; everything else stays plain text
  return esc(s)
    .replace(/```(\w*)\n([\s\S]*?)```/g, (_, l, c) => `<pre><code>${c}</code></pre>`)
    .replace(/`([^`\n]+)`/g, '<code>$1</code>')
    .replace(/\*\*([^*]+)\*\*/g, '<b>$1</b>');
}
function add(cls, html) {
  const d = document.createElement('div');
  d.className = 'msg ' + cls; d.innerHTML = html;
  log.appendChild(d); log.scrollTop = log.scrollHeight; return d;
}
function busy(b) { sendBtn.disabled = b; stopBtn.disabled = !b; input.disabled = b; }

async function refreshInfo() {
  const s = await (await fetch('status')).json();
  document.getElementById('info').textContent =
    `project: ${s.workdir}  ·  session: ${s.session_id ? s.session_id.slice(0, 8) : 'new'}`;
}

function render(ev) {
  if (ev.type === 'assistant' && ev.message) {
    for (const part of ev.message.content || []) {
      if (part.type === 'text' && part.text.trim()) add('bot', md(part.text));
      if (part.type === 'tool_use') {
        const i = part.input || {};
        const detail = i.command || i.file_path || i.pattern || i.url || i.description || '';
        add('tool', `⚙ ${esc(part.name)} ${esc(String(detail)).slice(0, 200)}`);
      }
    }
  } else if (ev.type === 'result') {
    if (ev.is_error) add('bot', '⚠ ' + esc(ev.result || 'error'));
    const cost = ev.total_cost_usd != null ? ` · $${ev.total_cost_usd.toFixed(4)}` : '';
    add('meta', `done in ${((ev.duration_ms || 0) / 1000).toFixed(1)}s${cost}`);
  } else if (ev.type === 'raw') {
    add('tool', esc(ev.text));
  }
}

async function send(text) {
  add('user', esc(text)); busy(true);
  try {
    const res = await fetch('chat', { method: 'POST', headers: {'Content-Type': 'application/json'},
                                      body: JSON.stringify({ message: text }) });
    if (!res.ok) { add('bot', '⚠ ' + esc((await res.json()).error)); return; }
    const reader = res.body.getReader(); const dec = new TextDecoder(); let buf = '';
    while (true) {
      const { value, done } = await reader.read(); if (done) break;
      buf += dec.decode(value, { stream: true });
      let i;
      while ((i = buf.indexOf('\n\n')) >= 0) {
        const chunk = buf.slice(0, i); buf = buf.slice(i + 2);
        if (chunk.startsWith('data: ')) render(JSON.parse(chunk.slice(6)));
      }
    }
  } catch (e) { add('bot', '⚠ ' + esc(String(e))); }
  finally { busy(false); refreshInfo(); input.focus(); }
}

document.getElementById('form').onsubmit = e => {
  e.preventDefault(); const t = input.value.trim(); if (!t) return; input.value = ''; send(t);
};
input.addEventListener('keydown', e => {
  if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); document.getElementById('form').requestSubmit(); }
});
stopBtn.onclick = () => fetch('stop', { method: 'POST' });
document.getElementById('reset').onclick = async () => {
  await fetch('reset', { method: 'POST' }); log.innerHTML = ''; refreshInfo();
};
refreshInfo(); input.focus();
</script>
</body>
</html>
```

### Step 4 — Create the launcher `~/claude-chat-ui/start.sh`

```bash
#!/usr/bin/env bash
# Usage: ./start.sh /path/to/project [port]
export CC_WORKDIR="${1:-$PWD}"
export CC_PORT="${2:-8765}"
# Permission mode: default | acceptEdits | plan | bypassPermissions
export CC_PERMISSION_MODE="${CC_PERMISSION_MODE:-acceptEdits}"
cd "$(dirname "$0")" && exec python3 server.py
```

```bash
chmod +x ~/claude-chat-ui/start.sh
```

### Step 5 — Start it

```bash
~/claude-chat-ui/start.sh /home/you/myproject
```

Expected output: `Claude Chat UI on http://127.0.0.1:8765  (project: /home/you/myproject)`.

### Step 6 — Open the chat window

Pick the case that matches your setup.

**A. VS Code runs on your PC and connects to the VM with Remote-SSH**

1. Start the server from the VS Code integrated terminal (`` Ctrl+` ``) using Step 5.
2. VS Code detects port 8765 and forwards it automatically. Check the **Ports** tab; add `8765` manually if it is missing.
3. `Ctrl+Shift+P` → **Simple Browser: Show** → `http://localhost:8765`.
4. Drag the Simple Browser tab to the right side. Now you have code on the left and chat on the right.

**B. VS Code runs on the VM itself (you use the VM desktop)**

1. Start the server in any terminal on the VM.
2. `Ctrl+Shift+P` → **Simple Browser: Show** → `http://127.0.0.1:8765`, or open that URL in the VM's browser.

**C. You only have SSH (VS Code opens separately on your PC)**

1. Open a tunnel from your PC:
   ```bash
   ssh -N -L 8765:127.0.0.1:8765 user@your-vm-ip
   ```
2. Open `http://localhost:8765` in your PC browser, or in VS Code via **Simple Browser: Show**.

### Step 7 — Use it

- **Enter** sends. **Shift+Enter** adds a new line.
- **Stop** kills the running Claude process.
- **New chat** starts a fresh session. Without it, every message continues the same conversation.
- Grey `⚙` lines show what Claude does: `Bash`, `Edit`, `Read`, and so on.
- Watch file changes appear live in the VS Code editor.

### Step 8 — Permissions (important)

`claude -p` cannot show interactive "Allow?" prompts, so you set the policy in advance with `CC_PERMISSION_MODE`:

| Mode | Behaviour |
|---|---|
| `plan` | Read-only. Claude plans but changes nothing. |
| `default` | Tools that need approval are refused. Claude reports what it wanted to do. |
| `acceptEdits` (default here) | File edits allowed. Risky bash commands still refused. |
| `bypassPermissions` | Everything allowed. Use only on a disposable VM. |

For finer control, add allowed tools to `<project>/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": ["Bash(npm test:*)", "Bash(git status)", "Bash(git diff:*)"],
    "deny":  ["Bash(rm -rf:*)", "Read(./.env)"]
  }
}
```

### Step 9 — Optional: one-click start from VS Code

Create `<project>/.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Claude Chat UI",
      "type": "shell",
      "command": "~/claude-chat-ui/start.sh ${workspaceFolder}",
      "isBackground": true,
      "problemMatcher": [],
      "presentation": { "reveal": "silent", "panel": "dedicated" }
    }
  ]
}
```

Run it with `Ctrl+Shift+P` → **Tasks: Run Task** → **Claude Chat UI**, then open Simple Browser.

### Step 10 — Optional: keep it running as a service

Create `~/.config/systemd/user/claude-chat.service`:

```ini
[Unit]
Description=Claude Chat UI

[Service]
ExecStart=%h/claude-chat-ui/start.sh %h/myproject
Restart=on-failure
Environment=PATH=%h/.local/bin:%h/.npm-global/bin:/usr/local/bin:/usr/bin:/bin

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now claude-chat
loginctl enable-linger "$USER"
```

Adjust `PATH` so it contains the folder from `which claude`.

---

## Option 3 — Terminal in an editor tab (zero setup)

Not a GUI, but much nicer than a separate command prompt.

1. In VS Code: `Ctrl+Shift+P` → **Terminal: Create New Terminal in Editor Area**.
2. Run `claude` there. It now sits as a full tab beside your files. Split it right with the editor split button.
3. Optional shortcut in `keybindings.json`:
   ```json
   {
     "key": "ctrl+alt+c",
     "command": "workbench.action.createTerminalEditor",
     "args": { "config": { "name": "Claude" } }
   }
   ```
4. Useful in-chat helpers: `/` lists commands, `@file` attaches a file, `Esc` interrupts, `/clear` resets.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `claude: command not found` in the UI | Set `CC_CLAUDE_BIN=$(which claude)` before `start.sh`, or fix `PATH` in the service file. |
| Page loads but replies never come | Run `claude -p "hi" --output-format stream-json --verbose` in the terminal. Fix any login or network error it prints. |
| `Error: --output-format=stream-json requires --verbose` | You edited the command. Keep `--verbose` in `server.py`. |
| "Claude is still working…" (409) | Wait, or press **Stop**. The UI handles one message at a time. |
| Claude says it was not allowed to run a command | Expected in `default` / `acceptEdits`. Add an allow rule (Step 8) or switch mode. |
| Port 8765 busy | `./start.sh /path/to/project 8877`, then use that port everywhere. |
| Simple Browser is blank | Use `localhost` exactly as forwarded in the **Ports** tab, or open the URL in a normal browser. |
| Conversation lost context | You pressed **New chat**, or the server restarted. Resume an old one in the terminal with `claude --resume`. |

## Security notes

- The server listens on `127.0.0.1` only. Do not change `CC_HOST` to `0.0.0.0`: anyone who can reach that port could run commands on your VM through Claude.
- Reach it only through VS Code port forwarding or an SSH tunnel.
- Use `bypassPermissions` only on a VM you can throw away.
