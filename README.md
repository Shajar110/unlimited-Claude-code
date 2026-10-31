# Unlimited Claude Code

Claude Code runs on your own **Anthropic subscription** as usual. When you hit your usage limit, it **automatically falls back to OmniRoute**, a local AI router that sends your work to other providers (Kiro, Copilot, Cline, Kimi, Gemini, …). Your work keeps going instead of stopping until the limit resets. At the same time, **RTK** cuts the tokens Claude spends on command output.

It is built from two open-source projects:

| Project | Repo | Role here |
|---|---|---|
| **RTK** | https://github.com/rtk-ai/rtk | A Claude Code hook that compresses the output of shell commands (`git status`, test runs, builds, …) before Claude reads it, so fewer tokens are used. |
| **OmniRoute** | https://github.com/cheaper-inference/OmniRoute | A local router on `http://localhost:20128` that talks to 300+ AI providers and falls back automatically between them. Used only after your Claude limit is hit. |

---

## 🤖 How to use this file

Open Claude Code in any folder and say:

> Read https://github.com/Shajar110/unlimited-Claude-code/blob/main/README.md and set everything up for me exactly as described.

Claude does the full setup. When it's done, it tells you what to do in the OmniRoute dashboard: connect your providers.

---

## How it works

```
                       ┌──────────────────────────────┐
  you ──► claude ──►   │ claude-smart (wrapper)       │
                       │  limit flag set?             │
                       └───────┬──────────────┬───────┘
                          no   │              │  yes
                               ▼              ▼
                    Anthropic API      OmniRoute (localhost:20128)
                  (your subscription)   ├─► Kiro / Copilot / Cline / Kimi …
                                        └─► auto-fallback between providers

  Usage limit hit ─► StopFailure hook writes the flag + desktop notification
  You type /exit ─► wrapper resumes the same chat (--continue) via OmniRoute
  Limit resets   ─► flag expires ─► claude goes back to Anthropic
```

- **One shared instance.** OmniRoute runs as a single systemd user service. RTK is a hook binary, not a daemon. With 2, 3 or 10 terminals running `claude`, there is still exactly one OmniRoute and no duplicate processes.
- **Background only.** OmniRoute starts at login with `--no-open --no-tray`. No browser window, no tray icon.
- **Your Claude login is never sent to OmniRoute.** In fallback mode the wrapper sets `ANTHROPIC_AUTH_TOKEN`, which overrides your subscription login, so OmniRoute never receives your Anthropic credentials.

---

# 🛠️ SETUP INSTRUCTIONS (for Claude Code)

> **Claude:** follow every step below in order on the user's machine. Use their real paths (do not hard-code `/home/<someone>`). Before modifying any existing file (`~/.claude/settings.json`, `~/.bashrc`, `~/.zshrc`), make a backup and **merge** into it, never overwrite it. Verify each step before moving on. Talk to the user in the language they use with you.
>
> These instructions target **Linux with systemd** (Ubuntu, Mint, Debian, Fedora, Arch …). On macOS, replace Step 3's systemd unit with `omniroute autostart enable` (launchd) and skip the `systemctl` commands in the scripts.

## Step 0: Check prerequisites

```bash
command -v claude && claude --version     # Claude Code must already be installed and logged in
command -v node && node -v                # Node.js (current LTS) for OmniRoute
command -v npm
command -v python3                        # used by the hook / status-line scripts
command -v curl
systemctl --user is-system-running        # should print "running" (or "degraded")
```

- If `node`/`npm` is missing, install a current Node.js LTS (for example via the distro package manager or nvm).
- Make sure the user is logged in to Claude Code with their **own Anthropic subscription** (`claude` → `/login`).
- Remember the full path of the real Claude binary. You need it in Step 4:
  ```bash
  REAL_CLAUDE=$(readlink -f "$(command -v claude)"); echo "$REAL_CLAUDE"
  ```
  If `claude` is already a shell function or alias, use `type -a claude` to find the real binary path, usually `~/.local/bin/claude`.

## Step 1: Install RTK and hook it into Claude Code

Repo: https://github.com/rtk-ai/rtk

```bash
# Official installer: downloads the release binary, verifies its SHA-256, installs to ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh
rtk --version

# Preview, then install the global Claude Code hook
rtk init -g --dry-run
rtk init -g --auto-patch </dev/null
```

`rtk init -g` adds a `PreToolUse` hook (`rtk hook claude`, matcher `Bash`) to `~/.claude/settings.json`, creates `~/.claude/RTK.md`, and adds `@RTK.md` to `~/.claude/CLAUDE.md`.

**Verify:**
```bash
echo '{"tool_name":"Bash","tool_input":{"command":"git status"}}' | rtk hook claude
# expected: ... "updatedInput":{"command":"rtk git status"} ...
```

> ⚠️ Do **not** set up `rtk-proxy` / `rtk claude`. That daemon also sits between Claude and the API, so it conflicts with this setup. If `rtk init` asks about the proxy, answer **no**. OmniRoute already has RTK compression built in (`omniroute ctx rtk`).

Notes:
- `~/.local/bin` must be on `PATH`. Check with `echo $PATH`.
- The hook only applies to Claude's **Bash** tool calls. Built-in Read/Grep/Glob are not affected.
- `rtk gain` shows how many tokens have been saved.

## Step 2: Install OmniRoute

Repo: https://github.com/cheaper-inference/OmniRoute

```bash
npm install -g omniroute        # use `sudo npm install -g omniroute` if global npm needs root
omniroute --version
```

OmniRoute keeps its data in `~/.omniroute/`. Its default address is `127.0.0.1:20128`.

> ⚠️ Do **not** run plain `omniroute` or `omniroute serve` in a terminal after Step 3. The service already owns port 20128, so you'd get *"Port 20128 is already in use"*. To see the dashboard, just open http://localhost:20128 in a browser.

## Step 3: Run OmniRoute in the background (one shared instance)

Don't use `omniroute autostart enable` on Linux: it may fall back to a desktop-autostart entry that runs with a tray icon. Create a proper systemd **user** service instead:

```bash
OMNI_JS=$(readlink -f "$(command -v omniroute)")     # e.g. /usr/lib/node_modules/omniroute/bin/omniroute.mjs
NODE_BIN=$(command -v node)
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/omniroute.service <<EOF
[Unit]
Description=OmniRoute AI proxy router
After=network-online.target graphical-session.target
Wants=network-online.target

[Service]
Type=notify
NotifyAccess=all
WatchdogSec=180
TimeoutStartSec=300
ExecStart=$NODE_BIN $OMNI_JS serve --no-open --no-tray
Restart=on-failure
RestartSec=5
Environment="PATH=$(dirname "$NODE_BIN"):$HOME/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/bin:/bin"
EnvironmentFile=-$HOME/.omniroute/.env

[Install]
WantedBy=default.target
EOF

rm -f ~/.config/autostart/omniroute.desktop          # avoid a second instance from desktop autostart
systemctl --user daemon-reload
systemctl --user enable --now omniroute.service
```

**Verify** (the first start can take ~10–60 s):
```bash
systemctl --user is-active omniroute.service           # active
ss -ltn | grep 20128                                    # LISTEN 127.0.0.1:20128
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:20128/   # 200 or 307
```

Handy commands for the user:
```bash
systemctl --user start omniroute      # start
systemctl --user stop omniroute       # stop
systemctl --user restart omniroute    # restart
systemctl --user status omniroute     # status
journalctl --user -u omniroute -f     # live logs
```

## Step 4: Create the fallback scripts

### 4a. The wrapper: `~/.local/bin/claude-smart`

Replace the `REAL_CLAUDE` value with the path from Step 0.

```bash
mkdir -p ~/.claude/fallback ~/.local/bin
cat > ~/.local/bin/claude-smart <<'EOF'
#!/usr/bin/env bash
# claude wrapper: Anthropic subscription by default, OmniRoute after a usage limit.
REAL_CLAUDE=${REAL_CLAUDE:-$HOME/.local/bin/claude}   # real Claude Code binary (NOT this wrapper)
FLAG=$HOME/.claude/fallback/omni-until               # contains epoch when fallback expires
OMNI_URL=${OMNI_URL:-http://127.0.0.1:20128}
OMNI_MODEL=${OMNI_MODEL:-auto/best-coding}            # OmniRoute model/combo to use on fallback
OMNI_KEY=${OMNI_KEY:-omniroute-local}                 # OmniRoute API key, only needed if REQUIRE_API_KEY=true

run_omni() {
  systemctl --user is-active --quiet omniroute.service || systemctl --user start omniroute.service
  # ANTHROPIC_AUTH_TOKEN overrides the subscription login, so the OAuth token never reaches OmniRoute
  ANTHROPIC_BASE_URL=$OMNI_URL ANTHROPIC_AUTH_TOKEN=$OMNI_KEY \
  ANTHROPIC_MODEL=$OMNI_MODEL ANTHROPIC_DEFAULT_OPUS_MODEL=$OMNI_MODEL \
  ANTHROPIC_DEFAULT_SONNET_MODEL=$OMNI_MODEL ANTHROPIC_DEFAULT_HAIKU_MODEL=$OMNI_MODEL \
  CLAUDE_CODE_SUBAGENT_MODEL=$OMNI_MODEL \
    exec "$REAL_CLAUDE" "$@"
}

# Drop an expired fallback flag
[ -f "$FLAG" ] && [ "$(date +%s)" -ge "$(cat "$FLAG" 2>/dev/null || echo 0)" ] && rm -f "$FLAG"

case "${CLAUDE_ROUTE:-auto}" in
  direct) exec "$REAL_CLAUDE" "$@" ;;
  omni)   run_omni "$@" ;;
esac

if [ -f "$FLAG" ]; then
  echo "⚡ Claude limit active until $(date -d @"$(cat "$FLAG")" '+%a %H:%M') → OmniRoute ($OMNI_MODEL)" >&2
  run_omni "$@"
fi

"$REAL_CLAUDE" "$@"
rc=$?
# The StopFailure hook set the flag during this session: continue it through OmniRoute
if [ -f "$FLAG" ]; then
  echo "⚡ Usage limit hit → resuming this conversation through OmniRoute..." >&2
  run_omni --continue
fi
exit $rc
EOF
```

### 4b. The limit hook: `~/.claude/fallback/on-limit.sh`

```bash
cat > ~/.claude/fallback/on-limit.sh <<'EOF'
#!/usr/bin/env bash
# StopFailure(rate_limit) hook: mark the OmniRoute fallback active until the limit resets.
[ -n "$ANTHROPIC_BASE_URL" ] && exit 0   # already on OmniRoute
D=$HOME/.claude/fallback
reset=$(python3 -I -c 'import json,sys
try:
  r=json.load(open(sys.argv[1]))
  t=[w.get("resets_at",0) for w in r.values() if isinstance(w,dict) and w.get("used_percentage",0)>=99]
  print(int(max(t)) if t else "")
except Exception: print("")' "$D/rate_limits.json")
[ -z "$reset" ] && reset=$(( $(date +%s) + 5*3600 ))
echo "$reset" > "$D/omni-until"
notify-send -a Claude "Claude usage limit reached" "Type /exit — it will resume via OmniRoute until $(date -d @"$reset" '+%a %H:%M')" 2>/dev/null
exit 0
EOF
```

### 4c. The status line: `~/.claude/fallback/statusline.sh`

It shows `Opus | 5h 62% | 7d 30%` (plus `via OmniRoute` in fallback mode). It also saves the exact limit-reset time, which the hook uses.

```bash
cat > ~/.claude/fallback/statusline.sh <<'EOF'
#!/usr/bin/env bash
# Status line: caches subscription rate_limits for on-limit.sh and shows usage.
python3 -I -c 'import json,sys,os
d=json.load(sys.stdin); r=d.get("rate_limits")
if r: json.dump(r,open(os.path.expanduser("~/.claude/fallback/rate_limits.json"),"w"))
out=[d.get("model",{}).get("display_name","")]
for k,n in (("five_hour","5h"),("seven_day","7d")):
  w=(r or {}).get(k)
  if w: out.append("%s %.0f%%" % (n, w.get("used_percentage",0)))
if os.environ.get("ANTHROPIC_BASE_URL"): out.append("via OmniRoute")
print(" | ".join(x for x in out if x))'
EOF

chmod +x ~/.local/bin/claude-smart ~/.claude/fallback/on-limit.sh ~/.claude/fallback/statusline.sh
bash -n ~/.local/bin/claude-smart && bash -n ~/.claude/fallback/on-limit.sh && echo syntax-ok
```

> If the user already has a custom `statusLine` in `~/.claude/settings.json`, ask before replacing it. The fallback still works without the status line; it then assumes a 5-hour reset window.

## Step 5: Register the hooks in `~/.claude/settings.json`

Back up the file, then **merge** these keys into it. Keep everything that's already there, including the RTK `PreToolUse` hook from Step 1:

```bash
cp ~/.claude/settings.json ~/.claude/settings.json.bak 2>/dev/null
python3 -I - <<'EOF'
import json, os
p = os.path.expanduser("~/.claude/settings.json")
s = json.load(open(p)) if os.path.exists(p) else {}
h = s.setdefault("hooks", {})
# Ensure the single shared OmniRoute is running whenever a Claude session starts
h["SessionStart"] = [x for x in h.get("SessionStart", []) if "omniroute" not in json.dumps(x)] + [{
  "hooks": [{"type": "command",
             "command": "systemctl --user is-active --quiet omniroute.service || systemctl --user start --no-block omniroute.service; exit 0"}]}]
# Detect the usage limit
h["StopFailure"] = [x for x in h.get("StopFailure", []) if "on-limit.sh" not in json.dumps(x)] + [{
  "matcher": "rate_limit",
  "hooks": [{"type": "command", "command": "~/.claude/fallback/on-limit.sh"}]}]
s.setdefault("statusLine", {"type": "command", "command": "~/.claude/fallback/statusline.sh"})
json.dump(s, open(p, "w"), indent=2)
print(json.dumps(s, indent=2))
EOF
```

Final `~/.claude/settings.json` should contain at least:

```json
{
  "hooks": {
    "PreToolUse":   [{ "matcher": "Bash", "hooks": [{ "type": "command", "command": "rtk hook claude" }] }],
    "SessionStart": [{ "hooks": [{ "type": "command", "command": "systemctl --user is-active --quiet omniroute.service || systemctl --user start --no-block omniroute.service; exit 0" }] }],
    "StopFailure":  [{ "matcher": "rate_limit", "hooks": [{ "type": "command", "command": "~/.claude/fallback/on-limit.sh" }] }]
  },
  "statusLine": { "type": "command", "command": "~/.claude/fallback/statusline.sh" }
}
```

## Step 6: Make `claude` use the wrapper

Add this to `~/.bashrc` (and `~/.zshrc` if the user uses zsh). Skip it if a `claude-smart` line is already there:

```bash
grep -q claude-smart ~/.bashrc || cat >> ~/.bashrc <<'EOF'

# Claude: Anthropic first, OmniRoute fallback after usage limit (see ~/.local/bin/claude-smart)
claude()        { ~/.local/bin/claude-smart "$@"; }
claude-direct() { CLAUDE_ROUTE=direct ~/.local/bin/claude-smart "$@"; }
claude-omni()   { CLAUDE_ROUTE=omni ~/.local/bin/claude-smart "$@"; }
EOF
```

| Command | What it does |
|---|---|
| `claude` | Smart: Anthropic first, OmniRoute after the limit |
| `claude-direct` | Always Anthropic |
| `claude-omni` | Always OmniRoute |
| `OMNI_MODEL=auto/pro-coding claude-omni` | OmniRoute with a different combo |

## Step 7: Check whether OmniRoute needs an API key

```bash
grep '^REQUIRE_API_KEY' ~/.omniroute/.env 2>/dev/null || echo "REQUIRE_API_KEY not set (key not required)"
```

- If it is `false` or not set, nothing more to do.
- If it is `true`, create an API key in the dashboard (**http://localhost:20128 → API Keys**). Then add `export OMNI_KEY=<that key>` to `~/.bashrc`. Don't paste the key into the Claude chat; the user should do this themselves.

## Step 8: Verify everything

```bash
# 1. RTK hook
echo '{"tool_name":"Bash","tool_input":{"command":"git status"}}' | rtk hook claude

# 2. OmniRoute service
systemctl --user is-active omniroute.service

# 3. Status line + limit hook (simulated, then cleaned up)
echo '{"model":{"display_name":"Opus"},"rate_limits":{"five_hour":{"used_percentage":100,"resets_at":'$(( $(date +%s)+3600 ))'}}}' | ~/.claude/fallback/statusline.sh
env -u ANTHROPIC_BASE_URL ~/.claude/fallback/on-limit.sh && date -d @"$(cat ~/.claude/fallback/omni-until)"
rm -f ~/.claude/fallback/omni-until ~/.claude/fallback/rate_limits.json
```

The **end-to-end OmniRoute test** (`claude-omni -p "Reply with exactly: fallback OK"`) only works **after the user has connected providers** (next section). Until then OmniRoute's free keyless providers usually fail with 429/402/403 errors. That's expected.

Once providers are connected, test with:
```bash
curl -s http://127.0.0.1:20128/v1/messages -H 'content-type: application/json' -H 'anthropic-version: 2023-06-01' \
  -d '{"model":"auto/best-coding","max_tokens":30,"messages":[{"role":"user","content":"Reply with exactly: OK working"}]}'
CLAUDE_ROUTE=omni ~/.local/bin/claude-smart -p "Reply with exactly: fallback OK"
```
A warning that Claude Code *"doesn't recognize the model auto/best-coding"* is harmless.

## Step 9: Final message to the user (Claude, say this in chat)

When all steps above are done, tell the user the following **in their language**:

> ✅ **Setup complete!**
>
> - **RTK** is active. Claude's command output is now compressed to save tokens. Run `rtk gain` any time to see the savings.
> - **OmniRoute** is running in the background at **http://localhost:20128**. It starts automatically at login and only one instance runs, however many terminals you open.
> - Open a **new terminal** (or run `source ~/.bashrc`) so the new `claude` command takes effect.
>
> **👉 One thing left for you: connect providers in OmniRoute**
>
> 1. Open **http://localhost:20128** in your browser.
> 2. Go to **Providers**.
> 3. Connect providers **one by one**. Click a provider, then **Connect / Sign in (OAuth)**, and log in with your account in the browser. For providers that use an API key, paste the key there in the dashboard, never in the Claude chat.
> 4. Connect **as many as you can** (e.g. Kiro, GitHub Copilot, Cline, Kilo Code, Kimi, Antigravity/Gemini, Groq, Mistral, OpenRouter …). The more providers you connect, the more fallback capacity you have. OmniRoute switches between them automatically when one is rate-limited.
>
> **⚠️ Do NOT connect "Claude Code (OAuth)" / your Claude account inside OmniRoute.** That would be your *same* Claude subscription with the *same* limit. When your Claude limit runs out, OmniRoute would hit the same exhausted account and you'd gain nothing. It also goes against Anthropic's usage policy and can get your Claude account restricted.
>
> **How it works from now on**
> - Use `claude` normally. It runs on your own Anthropic subscription.
> - When your usage limit is hit, you get a desktop notification. Type **`/exit`** and the same conversation automatically continues through OmniRoute.
> - Until your limit resets, every new `claude` you open goes through OmniRoute. After the reset it goes back to Anthropic by itself.
> - So your work doesn't stop when your Claude limit runs out.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Port 20128 is already in use` | Expected: the background service is already running. Don't run `omniroute` / `omniroute serve`; open http://localhost:20128 instead. |
| OmniRoute not running | `systemctl --user restart omniroute` then `journalctl --user -u omniroute -n 50` |
| Fallback didn't trigger when the limit hit | Type `/exit` and run `claude-omni -c` manually. Check `~/.claude/settings.json` has the `StopFailure` hook. |
| Stuck on OmniRoute after the limit reset | `rm ~/.claude/fallback/omni-until` |
| All requests via OmniRoute fail | No working provider is connected. Connect more in the dashboard, or try another combo: `OMNI_MODEL=auto/coding claude-omni` |
| `oauth start` says *Unknown provider* | Some providers only accept API keys (e.g. `gemini` = Google AI Studio key). Use the dashboard, or `omniroute providers available` to see each provider's auth type. |
| OAuth login in Claude's `!` prompt fails | `!` commands can't take interactive input. Run OAuth/login commands in a normal terminal, or use the dashboard. |
| Want to turn OmniRoute off for good | `systemctl --user disable --now omniroute` and remove the `SessionStart` hook |

## Uninstall

```bash
rtk init -g --uninstall                                   # remove the RTK hook
systemctl --user disable --now omniroute.service
rm ~/.config/systemd/user/omniroute.service
rm -rf ~/.claude/fallback ~/.local/bin/claude-smart
# then remove the claude()/claude-direct()/claude-omni() lines from ~/.bashrc
# and the SessionStart / StopFailure / statusLine entries from ~/.claude/settings.json
npm uninstall -g omniroute                                # optional
```

## Notes & limits

- A running Claude session can't switch endpoints mid-conversation, so one `/exit` is needed when the limit hits. The wrapper then resumes automatically with `--continue`.
- Claude Code's docs do not explicitly say that the subscription "usage limit reached" error is reported as `rate_limit`. If the hook doesn't fire for you, use `claude-omni` manually.
- Fallback models (MiniMax, Kimi, GLM, GPT, Gemini …) are capable, but their quality differs from Claude.
- Many OmniRoute providers are other tools' subscription accounts (Cursor, Kiro, Devin, Cline, Trae, Antigravity …), not official APIs. Using them through a proxy may break those services' terms and risks those accounts. API-key providers (Groq, Mistral, OpenRouter, Moonshot, AI Studio …) are the safest choice.
