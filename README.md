# OpenClaw: Rolling Step-by-Step Progress Setup — Complete Guide

**Goal:** Make OpenClaw show **live rolling progress for every tool/command step** in Telegram, exactly how you want it — with options for raw command text, natural narration, or labels only.

This guide is **verified end-to-end** on OpenClaw `2026.9.7` (Debian, Node v24.21). It covers the correct config structure, the exact keys, how to apply it, and the pitfalls that break a naive setup.

---

## 1. The one you were missing: `channels.telegram.streaming`

OpenClaw's Telegram streaming/progress is **not** a flat `mode` or a `progress` object at the top level. It lives under:

```json
channels.telegram.streaming
```

If you put `mode`/`progress` directly under `channels.telegram`, **it is silently ignored** and you see only "working…" with no detail. This is the #1 root cause of "kitna bhi bolo, steps nahi dikhte."

---

## 2. The exact config that does rolling steps

### A. Raw command text (see every exec/command)

```json
"channels": {
  "telegram": {
    "streaming": {
      "mode": "progress",
      "progress": {
        "toolProgress": true,
        "commandText": "raw"
      }
    }
  }
}
```

### B. Labels only (no raw command text) — clean

```json
"channels": {
  "telegram": {
    "streaming": {
      "mode": "progress",
      "progress": {
        "toolProgress": true,
        "commandText": "status"
      }
    }
  }
}
```

### C. Pure natural narration (no tool labels at all)

```json
"channels": {
  "telegram": {
    "streaming": {
      "mode": "progress",
      "progress": {
        "toolProgress": false,
        "commandText": "status",
        "commentary": true
      }
    }
  }
}
```

---

## 3. Key reference

| Key | Location | Values | Meaning |
|---|---|---|---|
| `mode` | `channels.telegram.streaming.mode` | `off \| partial \| block \| progress` | `progress` = one editable status draft; default |
| `toolProgress` | `streaming.progress.toolProgress` | `true \| false` | Show the rolling tool log (incl. failures). **Only matters in `progress` mode.** |
| `commandText` | `streaming.progress.commandText` | `status \| raw` | `status` = tool label only; `raw` = explicit command text |
| `commentary` | `streaming.progress.commentary` | `true \| false` | Show preambles/commentary as interleaved rows instead of a headline |
| `preview.toolProgress` | `streaming.preview.toolProgress` | `true \| false` | Reuse/limit the edited preview message in `partial`/`block` modes |
| `preview.commandText` | `streaming.preview.commandText` | `status \| raw` | Command detail inside preview lines (default `status`) |

**Notes**
- `partial` / `block` modes show tool progress **by default**; `progress` mode shows them **only** with `streaming.progress.toolProgress: true`.
- `streaming.mode: "off"` disables preview edits **and** suppresses generic tool/progress chatter. Approval prompts, media, and errors still deliver normally.
- Compact status, `replyToMode`, and quote replies can suppress tool-progress lines — set `replyToMode: "off"` (or `streaming.progress.toolProgress: false`) if quote replies hide your progress.

---

## 4. How to apply (two ways)

### Option A — `openclaw config set` (hot-reloads, recommended)

```bash
openclaw config set channels.telegram.streaming '{"mode":"progress","progress":{"toolProgress":true,"commandText":"raw"}}' --strict-json
```

**No restart needed.** Config-set hot-reloads automatically.

### Option B — Direct JSON edit (then reload)

Edit `~/.openclaw/openclaw.json` directly (Python/`jq`/editor), then **back up first**:

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d%H%M%S)
```

After writing the file, **reload the gateway** — direct edits do **not** hot-reload:

```bash
# IMPORTANT: find the actual gateway PID, never kill blind
pgrep -f "openclaw gateway"

# Send SIGUSR1 (dash sh rejects `kill -SIGUSR1 <pid>` — use `-s USR1`)
kill -s USR1 <GATEWAY_PID>
```

---

## 5. Pitfalls that break rolling progress

| # | Pitfall | Fix |
|---|---|---|
| 1 | Putting `mode`/`progress` at the **wrong level** (under `channels.telegram`) | It belongs under `channels.telegram.streaming` |
| 2 | `kill -SIGUSR1 <pid>` → `Illegal option -S` (dash/sh shell) | Use `kill -s USR1 <pid>` |
| 3 | Direct JSON edit forgets reload | Send `kill -s USR1 <gateway-pid>` |
| 4 | `modelPolicy.allow: ["*"]` → config invalid | Use per-provider wildcards: `["internai/*","openrouter/*", ...]` |
| 5 | Existing running turn keeps old tool list | New tool/config applies on the **next runtime build** — test with a fresh task |
| 6 | Reading config via wrong CLI (node split) | `export PATH=/usr/bin:$PATH` before any `openclaw config` CLI work; match the CLI to the gateway version |
| 7 | A simple reply (no tool) shows no tool-line | Expected — tool-progress lines only appear while a **tool runs** |

---

## 6. Verify it's active

```bash
export PATH=/usr/bin:$PATH
openclaw config validate                 # → "Config valid: ..."
python3 -c "import json;print(json.dumps(json.load(open('$HOME/.openclaw/openclaw.json'))['channels']['telegram']['streaming'],indent=2))"
```

Then send a **new task** that runs a tool (a command, a file read, etc.) and watch the rolling lines arrive.

---

## 7. Seeing more: `/verbose` and `/reasoning`

- **Completed tool output** (full detail) — per-turn: `/verbose full`
- **Internal model reasoning** — `/reasoning stream`
- These are **per-turn** settings; a running turn won't pick them up mid-flight.

---

## 8. Quick copy-paste (default good balance)

Reveal rolling steps with labels, no raw command spam:

```json
{
  "channels": {
    "telegram": {
      "streaming": {
        "mode": "progress",
        "progress": {
          "toolProgress": true,
          "commandText": "status",
          "commentary": true
        }
      }
    }
  }
}
```

---

*Verified on OpenClaw 2026.9.7 / gateway SIGUSR1 reload. Docs reference: `docs/channels/telegram/messaging.md`.*
