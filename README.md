[中文](README_CN.md) | # claudeop

Launch Claude Code CLI with **free OpenRouter models** — no Anthropic subscription required.

![demo](demo.jpg)

A tiny shell wrapper that:
- Fetches all free models from OpenRouter live
- Lets you pick one with a fuzzy selector (fzf)
- Sends you straight into Claude Code
- **Never** touches your normal `claude` command

```
claudeop        → pick a free model, launch Claude Code
claude          → your normal Anthropic subscription (unchanged)
```

---

## What This Does

Think of it like two remote controls for the same TV:

| Command | What It Does | Who Pays |
|---------|---------------|----------|
| `claude` | Claude Code via your Anthropic account | Your Anthropic subscription |
| `claudeop` | Claude Code via OpenRouter's free models | Free (OpenRouter) |

Both share the same Claude Code settings, plugins, MCP servers, and project config. The only difference is **which model answers your prompts**.

---

## Prerequisites

Before you start, you need these three things installed on your computer. Don't worry — they're all free.

### 1. Claude Code CLI

Claude Code is Anthropic's coding assistant that runs in your terminal.

**Install:**
```bash
# macOS — paste this in Terminal and press Enter
curl -fsSL https://storage.googleapis.com/claude-code/claude-code-installer.sh | sh
```

**Check it worked:**
```bash
claude --version
# Should print something like: 2.1.175
```

> If you already have Claude Code installed, skip this step.

---

### 2. An OpenRouter Account & API Key

OpenRouter is a service that gives you access to AI models (including Claude) — some for free.

1. Go to [openrouter.ai](https://openrouter.ai) and **create a free account**
2. Once logged in, go to [Settings → API Keys](https://openrouter.ai/settings/keys)
3. Click **"Create Key"**
4. Copy the key (it starts with `sk-or-v1-...`)
5. Save it somewhere — you'll need it in Step 3

---

### 3. Homebrew (macOS package manager)

Homebrew installs other tools for you. You likely already have it.

**Check:**
```bash
brew --version
# If it prints a version number, you have it — skip the install
```

**Install (if needed):**
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

---

## Installation

### Step 1: Install `jq` and `fzf`

These are tiny tools that claudeop needs. Install them with one command:

```bash
brew install jq fzf
```

### Step 2: Add Your OpenRouter API Key to Your Shell

Open your shell profile:

```bash
nano ~/.zshrc
```

Scroll to the **bottom** and add this line (replace `sk-or-v1-YOUR_KEY_HERE` with your actual key):

```bash
export OPENROUTER_API_KEY="sk-or-v1-YOUR_KEY_HERE"
```

Save and exit:
- Press `Ctrl + O`, then `Enter` to save
- Press `Ctrl + X` to exit

Then reload your shell:

```bash
source ~/.zshrc
```

**Verify:**
```bash
echo $OPENROUTER_API_KEY
# Should print your key starting with sk-or-v1-
```

### Step 3: Add the `claudeop` Wrapper

Open your shell profile again:

```bash
nano ~/.zshrc
```

Scroll to the **bottom** and paste this entire block:

```bash
# ── Claudeop: Launch Claude Code with free OpenRouter models ──────────
#   claudeop            → pick a free model, launch Claude Code
#   claudeop -m <id>    → skip picker, use specific model
#   claude              → Anthropic subscription (unchanged)

# Fetch :free models from OpenRouter on every launch
_claudeop_models() {
  curl -s "https://openrouter.ai/api/v1/models" \
    -H "Authorization: Bearer $OPENROUTER_API_KEY" \
    -H "Content-Type: application/json" \
    | jq -r '[.data[] | select(.id | test(":free"))] | sort_by(.id) | .[].id' 2>/dev/null
}

claudeop() {
  if [[ -z "$OPENROUTER_API_KEY" ]]; then
    echo "❌ OPENROUTER_API_KEY not set"
    return 1
  fi
  local model=""
  local skip_picker=0
  local args=("$@")
  while [[ $# -gt 0 ]]; do
    case "$1" in
      -m) model="$2"; skip_picker=1; shift 2 ;;
      --model) model="$2"; skip_picker=1; shift 2 ;;
      *) shift ;;
    esac
  done
  if [[ $skip_picker -eq 0 ]]; then
    local models="$( _claudeop_models )"
    if [[ -z "$models" ]]; then
      echo "❌ No models found on OpenRouter"; return 1
    fi
    model=$(echo "$models" | fzf --prompt="OpenRouter model → " --height=~40% --reverse --no-info)
    if [[ -z "$model" ]]; then
      echo "Cancelled"; return 0
    fi
  fi
  echo "🟢 OpenRouter → [$model]"
  ANTHROPIC_API_KEY="" \
  ANTHROPIC_BASE_URL="https://openrouter.ai/api" \
  ANTHROPIC_AUTH_TOKEN="$OPENROUTER_API_KEY" \
  CLAUDE_CODE_MODEL_OVERRIDE="$model" \
  claude "${args[@]}"
}
```

Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`), then reload:

```bash
source ~/.zshrc
```

### Step 4: Test It

```bash
claudeop
```

You should see a **fuzzy picker** listing free models. Use arrow keys to pick one, press `Enter`, and Claude Code launches with that model.

Look for the banner:
```
🟢 OpenRouter → [nex-agi/nex-n2-pro:free]
```

You're in! Type your prompt and go.

---

## Usage

### Normal (pick a model each time)
```bash
claudeop
```
A list of free models appears. Arrow keys to browse, type to filter, `Enter` to select.

### Use a specific model (skip the picker)
```bash
claudeop -m "google/gemma-3-27b-it:free"
```

### Your regular Claude (subscription)
```bash
claude
```
Exactly as before. Untouched.

---

## How It Works (For the Curious)

```
claudeop
  │
  ├─ Fetches free models from OpenRouter's API
  ├─ Shows them in fzf (a fuzzy selector)
  ├─ You pick one
  └─ Launches `claude` with these env vars (only for that session):
       ANTHROPIC_BASE_URL = https://openrouter.ai/api
       ANTHROPIC_AUTH_TOKEN = your OpenRouter key
       ANTHROPIC_API_KEY = "" (blanked to prevent conflict)
       CLAUDE_CODE_MODEL_OVERRIDE = the model you picked
```

Your shell environment is **never modified**. The env vars only exist inside that single Claude Code process. When it exits, they're gone.

---

## Troubleshooting

### "❌ OPENROUTER_API_KEY not set"
Your API key isn't in your shell. Go back to Step 2 and make sure you added the `export` line to `~/.zshrc` and ran `source ~/.zshrc`.

### "❌ No models found on OpenRouter"
- Check your internet connection
- Your API key might be invalid — try re-creating one at [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys)
- The `:free` filter might be too strict — temporarily test by running:
  ```bash
  curl -s https://openrouter.ai/api/v1/models \
    -H "Authorization: Bearer $OPENROUTER_API_KEY" \
    | jq '.data | length'
  ```
  If it prints a number > 0, your key works.

### The picker is empty / no models show
Free models come and go on OpenRouter. Try again later — new free models appear regularly.

### Claude Code shows auth conflict warnings
If you previously logged into Claude Code with your Anthropic account:
1. Run `/logout` inside Claude Code
2. Quit Claude Code
3. Try `claudeop` again

### Model doesn't work / errors during use
Free models have limitations. Some don't support tool use, long contexts, or Claude Code's advanced features. Try a different free model from the picker.

---

## Uninstall

Remove the `claudeop` block from your `~/.zshrc`:

```bash
nano ~/.zshrc
# Delete everything between and including:
# "# ── Claudeop: Launch Claude Code with free OpenRouter models ──────────"
# and the closing "}"
```

Then optionally remove the API key line:

```bash
# Delete this line too if you don't use OpenRouter elsewhere:
export OPENROUTER_API_KEY="sk-or-v1-..."
```

Reload:
```bash
source ~/.zshrc
```

Your normal `claude` command is unaffected.

---

## License

MIT