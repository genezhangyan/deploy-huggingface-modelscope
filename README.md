# deploy-huggingface-modelscope

A Claude Code skill for deploying AI projects to both **HuggingFace Spaces** and **ModelScope Studios** in one command — no more repeating the same steps or hitting the same pitfalls.

## What it does

When you say "部署", "deploy", "发布", or "push to both platforms", Claude will automatically:

1. Read your git remotes to detect the HuggingFace (`hf`) and ModelScope (`ms`) remotes
2. Push code to HuggingFace — done, it auto-restarts
3. Push code to ModelScope — then **manually trigger a restart via API** (this is the critical step most people miss)
4. Report the live URLs and expected warm-up time

Supports deploying to either or both platforms.

---

## Why this skill exists

### The ModelScope restart trap

On ModelScope, `git push` only updates files. It does **not** restart the running container. If your space has ever failed to start, a push will never fix it — the old broken container keeps running.

You must call the restart API after every push. This is not documented clearly anywhere.

### Bearer token authentication doesn't work

The obvious approach — putting your access token in the `Authorization: Bearer` header — returns `"user not logged in"` with a 200 status. No error, just silently fails.

The correct approach is a two-step cookie-based auth:
1. POST to `/api/v1/login` with your AccessToken to get session cookies
2. Extract the `csrf_token` cookie
3. PUT to `/api/v1/studio/<user>/<space>/restart` with both the cookies and `X-CSRF-Token` header

```bash
MS_TOKEN="ms-your-token-here"
MS_USER="your-username"
MS_SPACE="your-space-name"

curl -s -X POST "https://www.modelscope.cn/api/v1/login" \
  -H "Content-Type: application/json" \
  -c /tmp/ms_cookies.txt \
  -d "{\"AccessToken\": \"$MS_TOKEN\"}"

CSRF=$(grep csrf_token /tmp/ms_cookies.txt | awk '{print $NF}')

curl -s -X PUT "https://www.modelscope.cn/api/v1/studio/${MS_USER}/${MS_SPACE}/restart" \
  -b /tmp/ms_cookies.txt \
  -H "X-CSRF-Token: $CSRF" \
  -H "Content-Type: application/json"
```

### Don't put pre-installed packages in requirements.txt

ModelScope pre-installs `torch` (2.3.1) and `gradio` (6.2.0). If you pin those in `requirements.txt`, pip will try to reinstall them — and fail, breaking the entire startup sequence with cryptic errors.

Safe `requirements.txt` for a typical ModelScope AI space:
```
Pillow>=10.0.0
numpy>=1.26.0
huggingface_hub>=0.27.0
```

Do **not** add: `torch`, `gradio`, `modelscope-studio`

---

## Known issues & fixes

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Push succeeds but site doesn't update | Push doesn't trigger restart | Call restart API after every push |
| `restart` returns "user not logged in" | Bearer token auth doesn't work | Use cookie-based auth (see above) |
| Space fails to start after push | `requirements.txt` conflicts with pre-installed packages | Remove torch/gradio from requirements.txt |
| `git push` authentication error | Token expired | Reconfigure git credentials |
| Space starts but API errors | Port not configured | Set `server_port=7860` explicitly (PORT env var is not set) |

---

## Installation

Copy `SKILL.md` to your Claude skills directory:

```bash
# macOS / Linux
cp SKILL.md ~/.claude/skills/deploy-huggingface-modelscope/SKILL.md

# Windows (Git Bash)
cp SKILL.md /c/Users/<you>/.claude/skills/deploy-huggingface-modelscope/SKILL.md
```

Claude Code picks up new skills automatically on next session start.

---

## Setup for your project

### 1. Add git remotes

```bash
# HuggingFace
git remote add hf https://huggingface.co/spaces/<username>/<space-name>

# ModelScope
git remote add ms https://www.modelscope.cn/studios/<username>/<space-name>.git
```

### 2. Save your ModelScope token

Add to your project's `CLAUDE.md` or `.claude/memory/MEMORY.md` so Claude can find it without asking every time:

```markdown
## ModelScope
- AccessToken: ms-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
- Username: your-username
- Space: your-space-name
```

Get your token: https://www.modelscope.cn → Profile → Access Token

---

## Usage

Once installed, just tell Claude what you want:

- `"部署"` / `"deploy"` → pushes to both platforms + triggers MS restart
- `"只部署 HuggingFace"` → HF only
- `"只部署 ModelScope"` → MS push + restart
- `"发布最新版本"` → same as deploy

---

## Platform reference

| | HuggingFace Spaces | ModelScope Studios |
|---|---|---|
| Auto-restart on push | ✅ Yes | ❌ No — must call API |
| Auth method | Standard git | Cookie-based (see above) |
| Pre-installed packages | Varies by runtime | torch 2.3.1, gradio 6.2.0 |
| Warm-up time | 1–3 min | 2–5 min (longer if lazy-installing deps) |
| Default port | Auto | Must set `server_port=7860` |
