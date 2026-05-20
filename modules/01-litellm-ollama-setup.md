# Module 01 -- LiteLLM + Ollama Setup

Estimated time: 20 minutes
Prerequisites: macOS with zsh, Homebrew, ~20 GB free disk

Install Ollama, configure a LiteLLM proxy, and create the `claude-litellm` profile so Claude Code runs against a local model with zero cloud dependencies.

## Orientation

Print this once at the start:

```
You're setting up a local LLM backend for Claude Code.
This takes about 20 minutes (plus model download time).

We'll set up:
  1. Ollama (local model runtime)
  2. A local model (qwen3-coder:30b)
  3. LiteLLM proxy (translates Anthropic API calls to Ollama)
  4. claude-litellm shell profile (third profile alongside claude-work and claude-personal)

When done, you'll have three Claude Code profiles:
  claude-work     -> Vertex AI (Red Hat work, cloud)
  claude-personal -> Anthropic API (personal, cloud)
  claude-litellm  -> Ollama via LiteLLM (local, offline, AUP-safe)

You'll need:
  - macOS with zsh
  - Homebrew installed
  - ~20 GB free disk space (for model weights)
  - Your claude-work and/or claude-personal profiles already working
    (see the Vertex AI course Module 01 for setup, but not strictly required)
```

## Progress Tracking

On module start, write a progress marker:

```bash
mkdir -p ~/.claude/courseware-progress && date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/01.started
```

## Preflight

Audit current state before doing anything:

```bash
# 1. macOS check
[ "$(uname -s)" = "Darwin" ] && echo "EXISTS: macOS ($(sw_vers -productVersion 2>/dev/null || uname -r))" || echo "MISSING: macOS required (found $(uname -s))"

# 2. zsh check
[ "$(basename "$SHELL")" = "zsh" ] && echo "EXISTS: zsh default shell" || echo "MISSING: zsh (found $SHELL)"

# 3. Homebrew
command -v brew &>/dev/null && echo "EXISTS: Homebrew ($(brew --version 2>/dev/null | head -1))" || echo "MISSING: Homebrew"

# 4. Disk space (~20 GB needed)
AVAIL_GB=$(df -g "$HOME" 2>/dev/null | tail -1 | awk '{print $4}')
if [ -z "$AVAIL_GB" ]; then
  AVAIL_GB=$(df -BG "$HOME" 2>/dev/null | tail -1 | awk '{print $4}' | tr -d 'G')
fi
[ "${AVAIL_GB:-0}" -ge 20 ] && echo "EXISTS: ${AVAIL_GB} GB free disk space" || echo "MISSING: need ~20 GB free (found ${AVAIL_GB:-unknown} GB)"

# 5. Ollama
command -v ollama &>/dev/null && echo "EXISTS: Ollama ($(ollama --version 2>/dev/null || echo 'installed'))" || echo "MISSING: Ollama"

# 6. Ollama models (if installed)
if command -v ollama &>/dev/null; then
  MODELS=$(ollama list 2>/dev/null | tail -n +2)
  if [ -n "$MODELS" ]; then
    echo "EXISTS: Ollama models:"
    echo "$MODELS" | while read -r line; do echo "  $line"; done
  else
    echo "MISSING: No Ollama models downloaded"
  fi
fi

# 7. uv
command -v uv &>/dev/null && echo "EXISTS: uv ($(uv --version 2>/dev/null))" || echo "MISSING: uv"

# 8. LiteLLM config
[ -f "$HOME/.config/litellm/config.yaml" ] && echo "EXISTS: LiteLLM config at ~/.config/litellm/config.yaml" || echo "MISSING: LiteLLM config"

# 9. LiteLLM .env
[ -f "$HOME/.config/litellm/.env" ] && echo "EXISTS: LiteLLM .env at ~/.config/litellm/.env" || echo "MISSING: LiteLLM .env"

# 10. litellm-proxy launcher
[ -x "$HOME/.local/bin/litellm-proxy" ] && echo "EXISTS: litellm-proxy launcher" || echo "MISSING: litellm-proxy launcher"

# 11. claude-litellm profile
grep -q "claude-litellm" "$HOME/.zshrc" 2>/dev/null && echo "EXISTS: claude-litellm profile in ~/.zshrc" || echo "MISSING: claude-litellm profile"

# 12. Existing profiles (informational)
grep -q "claude-work" "$HOME/.zshrc" 2>/dev/null && echo "EXISTS: claude-work profile" || echo "INFO: claude-work profile not found (optional)"
grep -q "claude-personal" "$HOME/.zshrc" 2>/dev/null && echo "EXISTS: claude-personal profile" || echo "INFO: claude-personal profile not found (optional)"

# 13. No conflicting ANTHROPIC_API_KEY export
if grep -q "^export ANTHROPIC_API_KEY" "$HOME/.zshrc" 2>/dev/null; then
  echo "WARNING: ANTHROPIC_API_KEY exported in ~/.zshrc -- this can interfere with profile switching"
else
  echo "EXISTS: No global ANTHROPIC_API_KEY export (good)"
fi

# 14. jq (used in verification)
command -v jq &>/dev/null && echo "EXISTS: jq ($(jq --version 2>/dev/null))" || echo "MISSING: jq (optional, used for JSON parsing in verification)"
```

Print a summary of what was found. Skip any step below where the item already exists and is valid.

## Step 1 -- Choose Your Path

This module offers two setup paths:

```
Two options:

  Option A (RECOMMENDED): Automated via Claude Code
    Paste a prompt into Claude Code and it handles everything.
    Faster, less error-prone, same result.

  Option B: Manual steps
    Run each command yourself. Good for understanding every piece.

Which do you prefer? (A or B)
```

If the user chooses Option A, go to Step 2A. If Option B, go to Step 2B.

## Step 2A -- Automated Setup via Claude Code

Tell the user:

```
Open a NEW Claude Code session using your work or personal profile:

  claude-work
  or
  claude-personal

Then paste this entire prompt:
```

Embed this prompt verbatim inside a fenced code block:

```
Set up a local LLM environment for Claude Code that routes Claude Code through a
local LiteLLM proxy to a LOCAL Ollama backend ONLY. Follow this plan
strictly: discover, propose, wait for approval, implement, verify.

HARD CONSTRAINTS -- do not deviate
- Backend = local Ollama ONLY. No OpenAI, no Gemini, no Anthropic
  passthrough, no Ollama Cloud, no remote endpoints of any kind.
- The LiteLLM config must NOT contain entries for any non-Ollama
  provider, even commented out.
- The proxy must bind to 127.0.0.1 only -- never 0.0.0.0.
- Structure the config so a future module can add another LOCAL model
  by appending one `model_list` entry. No external-provider scaffolding.
- Do NOT install LiteLLM PyPI 1.82.7 or 1.82.8 (compromised). Pin >= 1.83.
- Do not touch `claude-work` or `claude-personal`.

PHASE 1 -- Discover (read-only, report back)
  1. Find existing `claude-work` and `claude-personal` definitions in
     ~/.zshrc and any sourced files. Show me the exact lines and style.
  2. Confirm shell is zsh on macOS; locate Homebrew prefix.
  3. Report install status of: `ollama`, `uv`, `litellm`, `jq`.
  4. If `ollama` is installed, run `ollama list` and report what's there.
  5. Check available disk space in $HOME (need ~20 GB headroom).
  STOP. Show me findings before proceeding.

PHASE 2 -- Propose (wait for "approved")
  a. Install plan: `ollama` (if missing) via Homebrew, `uv` via Homebrew,
     then `ollama pull qwen3-coder:30b` (default model).
  b. `~/.config/litellm/config.yaml` with a SINGLE model entry:
       model_list:
         - model_name: qwen3-coder
           litellm_params:
             model: ollama_chat/qwen3-coder:30b
             api_base: http://127.0.0.1:11434
     plus comment: "# To add another LOCAL model later, append a new
     entry here. Do NOT add non-Ollama providers -- this profile is
     local-only by policy."
     `general_settings.master_key: os.environ/LITELLM_MASTER_KEY`
     `litellm_settings.drop_params: true`
  c. `~/.config/litellm/.env` (chmod 600) with a freshly generated
     `LITELLM_MASTER_KEY=sk-litellm-<random>` and
     `LITELLM_LOCAL_MODEL_COST_MAP=true`.
  d. `~/.local/bin/litellm-proxy` launcher: `uv tool run litellm
     --config ~/.config/litellm/config.yaml --host 127.0.0.1 --port 4000`,
     sourcing the .env first. chmod +x.
  e. zsh function `claude-litellm` placed right after the other two
     profile functions in ~/.zshrc, matching their comment style:
       - source ~/.config/litellm/.env
       - check proxy reachable on 127.0.0.1:4000, error clearly if not
       - unset CLAUDE_CODE_USE_VERTEX, CLOUD_ML_REGION,
         ANTHROPIC_VERTEX_PROJECT_ID, ANTHROPIC_API_KEY
       - export ANTHROPIC_BASE_URL=http://127.0.0.1:4000
       - export ANTHROPIC_AUTH_TOKEN="$LITELLM_MASTER_KEY"
       - exec claude --model qwen3-coder "$@"
  f. Back up ~/.zshrc to ~/.zshrc.bak.$(date +%s) before editing.

PHASE 3 -- Implement (only after I say "approved")
  Execute the proposal exactly. Confirm before each `brew install` /
  `ollama pull` / `uv tool install`. qwen3-coder:30b pull is ~19 GB --
  warn me and wait.

PHASE 4 -- Verify
  1. Show diff of ~/.zshrc.
  2. Start `litellm-proxy` in background, capture PID.
  3. Smoke test:
       curl -s http://127.0.0.1:4000/v1/messages \
         -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
         -H "Content-Type: application/json" \
         -d '{"model":"qwen3-coder","max_tokens":32,
              "messages":[{"role":"user","content":"reply with the single word: ok"}]}'
  4. Confirm response, then kill the proxy.
  5. Print final usage summary:
     - claude-work     -> Vertex (Red Hat work)
     - claude-personal -> personal Claude
     - claude-litellm  -> local Ollama via LiteLLM (offline / experimental)
     Plus one-line commands to start/stop the proxy.
```

After the user pastes and completes the automated setup, skip to the Verification section.

## Step 2B -- Manual Setup

### Step 2B.1 -- Install Ollama

Skip if Ollama is already installed.

```
Ollama runs local models. Install it via Homebrew:

  ! brew install ollama
```

Verify:
```bash
command -v ollama && ollama --version
```

### Step 2B.2 -- Start Ollama and Pull a Model

Start the Ollama service if not running:

```
  ! ollama serve &
```

Wait a few seconds, then pull the default model (~19 GB download):

```
This pulls qwen3-coder:30b (~19 GB). Make sure you have enough disk space and a
reasonable connection. This will take several minutes.

  ! ollama pull qwen3-coder:30b
```

Verify:
```bash
ollama list | grep qwen3-coder
```

### Step 2B.3 -- Install uv

Skip if uv is already installed.

```
uv is a fast Python package manager. We use it to run LiteLLM without
a permanent global install.

  ! brew install uv
```

Verify:
```bash
command -v uv && uv --version
```

### Step 2B.4 -- Install jq

Skip if jq is already installed.

```
jq is used for parsing JSON responses in verification steps.

  ! brew install jq
```

Verify:
```bash
command -v jq && jq --version
```

### Step 2B.5 -- Configure LiteLLM

Create the LiteLLM config directory and files:

```bash
mkdir -p ~/.config/litellm
```

Write the proxy config:

```bash
cat > ~/.config/litellm/config.yaml << 'EOF'
# LiteLLM proxy config -- LOCAL Ollama backend ONLY
# To add another LOCAL model later, append a new entry here.
# Do NOT add non-Ollama providers -- this profile is local-only by policy.

model_list:
  - model_name: qwen3-coder
    litellm_params:
      model: ollama_chat/qwen3-coder:30b
      api_base: http://127.0.0.1:11434

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY

litellm_settings:
  drop_params: true
EOF
```

Generate a master key and write the .env file:

```bash
LITELLM_KEY="sk-litellm-$(openssl rand -hex 16)"
cat > ~/.config/litellm/.env << EOF
LITELLM_MASTER_KEY=$LITELLM_KEY
LITELLM_LOCAL_MODEL_COST_MAP=true
EOF
chmod 600 ~/.config/litellm/.env
echo "Master key generated and saved to ~/.config/litellm/.env"
```

Verify:
```bash
[ -f "$HOME/.config/litellm/config.yaml" ] && echo "PASS: config.yaml exists" || echo "FAIL: config.yaml missing"
[ -f "$HOME/.config/litellm/.env" ] && echo "PASS: .env exists" || echo "FAIL: .env missing"
PERMS=$(stat -f "%Lp" "$HOME/.config/litellm/.env" 2>/dev/null || stat -c "%a" "$HOME/.config/litellm/.env" 2>/dev/null)
[ "$PERMS" = "600" ] && echo "PASS: .env permissions 600" || echo "WARN: .env permissions are $PERMS (expected 600)"
```

### Step 2B.6 -- Create the Proxy Launcher

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/litellm-proxy << 'LAUNCHER'
#!/usr/bin/env zsh
# LiteLLM proxy launcher -- local Ollama backend only
source ~/.config/litellm/.env
exec uv tool run --from "litellm>=1.83" litellm \
  --config ~/.config/litellm/config.yaml \
  --host 127.0.0.1 \
  --port 4000
LAUNCHER
chmod +x ~/.local/bin/litellm-proxy
```

Verify:
```bash
[ -x "$HOME/.local/bin/litellm-proxy" ] && echo "PASS: litellm-proxy launcher is executable" || echo "FAIL: litellm-proxy not found or not executable"
```

### Step 2B.7 -- Create the claude-litellm Profile

Back up ~/.zshrc first:

```bash
cp ~/.zshrc ~/.zshrc.bak.$(date +%s)
echo "Backup saved"
```

Add the claude-litellm function to ~/.zshrc. Find the existing claude-work and claude-personal definitions and add this right after them, matching the comment style:

```bash
cat >> ~/.zshrc << 'PROFILE'

# Claude Code -- local Ollama via LiteLLM (offline / experimental)
claude-litellm() {
  source ~/.config/litellm/.env
  if ! curl -sf http://127.0.0.1:4000/health --max-time 2 >/dev/null 2>&1; then
    echo "LiteLLM proxy not reachable on 127.0.0.1:4000"
    echo "Start it first:  litellm-proxy"
    echo "  (or in background:  litellm-proxy &)"
    return 1
  fi
  (
    unset CLAUDE_CODE_USE_VERTEX CLOUD_ML_REGION ANTHROPIC_VERTEX_PROJECT_ID ANTHROPIC_API_KEY
    export ANTHROPIC_BASE_URL=http://127.0.0.1:4000
    export ANTHROPIC_AUTH_TOKEN="$LITELLM_MASTER_KEY"
    exec claude --model qwen3-coder "$@"
  )
}
PROFILE
```

Reload the shell config:

```bash
source ~/.zshrc
```

Verify:
```bash
grep -q "claude-litellm" "$HOME/.zshrc" && echo "PASS: claude-litellm in ~/.zshrc" || echo "FAIL: claude-litellm not found in ~/.zshrc"
```

## Verification

Run the full check:

```bash
PASS=0
TOTAL=8

# 1. Ollama installed
command -v ollama &>/dev/null && { echo "PASS: Ollama installed"; PASS=$((PASS+1)); } || echo "FAIL: Ollama not found"

# 2. qwen3-coder model
ollama list 2>/dev/null | grep -q "qwen3-coder" && { echo "PASS: qwen3-coder model available"; PASS=$((PASS+1)); } || echo "FAIL: qwen3-coder not found"

# 3. uv installed
command -v uv &>/dev/null && { echo "PASS: uv installed"; PASS=$((PASS+1)); } || echo "FAIL: uv not found"

# 4. LiteLLM config
[ -f "$HOME/.config/litellm/config.yaml" ] && { echo "PASS: LiteLLM config exists"; PASS=$((PASS+1)); } || echo "FAIL: LiteLLM config missing"

# 5. LiteLLM .env with correct permissions
if [ -f "$HOME/.config/litellm/.env" ]; then
  PERMS=$(stat -f "%Lp" "$HOME/.config/litellm/.env" 2>/dev/null || stat -c "%a" "$HOME/.config/litellm/.env" 2>/dev/null)
  [ "$PERMS" = "600" ] && { echo "PASS: LiteLLM .env (permissions 600)"; PASS=$((PASS+1)); } || { echo "PASS: LiteLLM .env (permissions $PERMS -- consider chmod 600)"; PASS=$((PASS+1)); }
else
  echo "FAIL: LiteLLM .env missing"
fi

# 6. Proxy launcher
[ -x "$HOME/.local/bin/litellm-proxy" ] && { echo "PASS: litellm-proxy launcher"; PASS=$((PASS+1)); } || echo "FAIL: litellm-proxy launcher missing"

# 7. claude-litellm profile
grep -q "claude-litellm" "$HOME/.zshrc" 2>/dev/null && { echo "PASS: claude-litellm profile"; PASS=$((PASS+1)); } || echo "FAIL: claude-litellm not in ~/.zshrc"

# 8. No conflicting ANTHROPIC_API_KEY
if grep -q "^export ANTHROPIC_API_KEY" "$HOME/.zshrc" 2>/dev/null; then
  echo "WARN: ANTHROPIC_API_KEY exported globally (may interfere)"
else
  echo "PASS: No conflicting ANTHROPIC_API_KEY"; PASS=$((PASS+1))
fi

echo ""
echo "$PASS/$TOTAL checks passed."
```

If all 8 pass:
```
All checks passed. Your local LLM environment is configured.

To use it:
  1. Start the proxy:     litellm-proxy     (or: litellm-proxy &)
  2. Launch Claude Code:  claude-litellm
  3. Stop the proxy:      kill $(lsof -ti:4000)   (or fg + Ctrl+C)

Your three profiles:
  claude-work     -> Vertex AI (Red Hat work)
  claude-personal -> personal Anthropic account
  claude-litellm  -> local Ollama via LiteLLM (offline / AUP-safe)
```

If any fail, tell the user which step to re-run.

### Smoke Test

After verification passes, run a live smoke test if the proxy is reachable:

```bash
source ~/.config/litellm/.env
if curl -sf http://127.0.0.1:4000/health --max-time 2 >/dev/null 2>&1; then
  echo "Proxy is running. Sending test request..."
  RESPONSE=$(curl -s http://127.0.0.1:4000/v1/messages \
    -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
    -H "Content-Type: application/json" \
    -d '{"model":"qwen3-coder","max_tokens":32,"messages":[{"role":"user","content":"reply with the single word: ok"}]}')
  echo "$RESPONSE" | jq -r '.content[0].text // .choices[0].message.content // "No parseable response"' 2>/dev/null || echo "$RESPONSE"
else
  echo "Proxy not running. Start it first to run the smoke test:"
  echo "  litellm-proxy &"
  echo "  sleep 5"
  echo "Then re-run this verification step."
fi
```

## Challenge

```
Now let's verify the full pipeline and explore model switching.

1. Pull one additional local model of your choice. Some suggestions:
     ollama pull llama3.1:8b       (~4.7 GB, general purpose)
     ollama pull codellama:13b     (~7.4 GB, code-focused)
     ollama pull mistral:7b        (~4.1 GB, general purpose)

2. Add it to your LiteLLM config as a second model_list entry.
   Edit ~/.config/litellm/config.yaml and append:

     - model_name: YOUR_MODEL_ALIAS
       litellm_params:
         model: ollama_chat/YOUR_MODEL:TAG
         api_base: http://127.0.0.1:11434

3. Restart the proxy (kill the old one, start fresh).

4. Start a claude-litellm session and switch to the new model:
     claude-litellm
     Then inside Claude Code: /model YOUR_MODEL_ALIAS

5. Ask the same question to both models and compare the responses.

Tell me:
  1. Which model did you add?
  2. How did the responses compare?
  3. What are the tradeoffs you noticed (speed, quality, size)?
```

## Challenge Verification

The user should report:

1. They successfully pulled a second model and it appears in `ollama list`
2. They added it to `~/.config/litellm/config.yaml` as a new `model_list` entry
3. They were able to switch between models using `/model` inside a `claude-litellm` session
4. They can articulate a tradeoff (e.g., smaller models are faster but less capable)

Verify programmatically:

```bash
# Check at least 2 models in Ollama
MODEL_COUNT=$(ollama list 2>/dev/null | tail -n +2 | wc -l | tr -d ' ')
[ "$MODEL_COUNT" -ge 2 ] && echo "PASS: $MODEL_COUNT models in Ollama" || echo "FAIL: only $MODEL_COUNT model(s) found"

# Check at least 2 entries in LiteLLM config
ENTRY_COUNT=$(grep -c "model_name:" "$HOME/.config/litellm/config.yaml" 2>/dev/null)
[ "$ENTRY_COUNT" -ge 2 ] && echo "PASS: $ENTRY_COUNT entries in LiteLLM config" || echo "FAIL: only $ENTRY_COUNT entry in config"
```

If successful, write the completion marker:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/01.done
```

Then print:
```
Module 01 complete.

Your local LLM environment is fully configured with multiple models.

Your profiles:
  claude-work     -> Vertex AI (cloud, full Claude capability)
  claude-personal -> Anthropic API (cloud, personal account)
  claude-litellm  -> Ollama via LiteLLM (local, offline, AUP-safe)

Start the proxy:     litellm-proxy
Launch local Claude:  claude-litellm
Stop the proxy:      kill $(lsof -ti:4000)

Questions or feedback? Open an issue in this repo.
```

<!-- NEW -->
