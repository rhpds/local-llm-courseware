# Module 02 -- System Prompts and Temperature

Estimated time: 10 minutes (5 minutes click-to-paste)
Prerequisites: Module 01 complete or OpenShift lab environment

Send the same question with different system prompts and temperature settings to see how they shape model output.

## Orientation

Print this once at the start:

```
You're learning to control LLM output.
This takes about 10 minutes.

We'll explore:
  1. System prompts (steering the model's behavior)
  2. Temperature (controlling randomness)

You'll need: LiteLLM running (locally or on OpenShift)
```

## Progress Tracking

On module start, write a progress marker:

```bash
mkdir -p ~/.claude/courseware-progress && date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/02.started
```

## Preflight

Audit current state before doing anything:

```bash
# 1. LiteLLM environment file
[ -f "$HOME/.config/litellm/.env" ] && echo "EXISTS: ~/.config/litellm/.env" || echo "INFO: No local .env (OK if using OpenShift)"

# 2. llm-env helper
[ -f "$HOME/.local/bin/llm-env" ] && echo "EXISTS: llm-env helper" || echo "MISSING: llm-env helper (will create in Step 1)"

# 3. LiteLLM reachable
source "$HOME/.config/litellm/.env" 2>/dev/null
LLM_URL="${LITELLM_URL:-http://127.0.0.1:4000}"
LLM_KEY="${LITELLM_API_KEY:-$LITELLM_MASTER_KEY}"
curl -sf "$LLM_URL/health" --max-time 5 &>/dev/null && echo "EXISTS: LiteLLM responding at $LLM_URL" || echo "MISSING: LiteLLM not reachable at $LLM_URL"

# 4. Available models
if curl -sf "$LLM_URL/health" --max-time 5 &>/dev/null; then
  MODELS=$(curl -s "$LLM_URL/v1/models" -H "Authorization: Bearer $LLM_KEY" --max-time 5 2>/dev/null | jq -r '.data[].id' 2>/dev/null)
  [ -n "$MODELS" ] && echo "EXISTS: Models: $MODELS" || echo "MISSING: No models found"
fi

# 5. jq
command -v jq &>/dev/null && echo "EXISTS: jq ($(jq --version 2>/dev/null))" || echo "MISSING: jq (install: brew install jq)"
```

Print a summary. If LiteLLM is not reachable, the user needs to either:
- Start the proxy locally: `litellm-proxy &` (wait ~10 seconds for startup)
- Or set `LITELLM_URL` and `LITELLM_API_KEY` environment variables for OpenShift

## Step 1 -- Create the Environment Helper

Skip if `~/.local/bin/llm-env` already exists.

This helper script discovers LiteLLM connection details so every subsequent snippet stays clean. All modules from here on source it.

```
Creating the llm-env helper so all API snippets stay short.

  ! mkdir -p ~/.local/bin && cat > ~/.local/bin/llm-env << 'HELPER'
source "$HOME/.config/litellm/.env" 2>/dev/null
export LLM_URL="${LITELLM_URL:-http://127.0.0.1:4000}"
export LLM_KEY="${LITELLM_API_KEY:-$LITELLM_MASTER_KEY}"
export LLM_MODEL=$(curl -s "$LLM_URL/v1/models" \
  -H "Authorization: Bearer $LLM_KEY" --max-time 5 2>/dev/null \
  | jq -r '.data[0].id // "qwen3-coder"' 2>/dev/null)
HELPER
  ! chmod +x ~/.local/bin/llm-env
```

Verify:

```bash
[ -f "$HOME/.local/bin/llm-env" ] && echo "PASS: llm-env created" || echo "FAIL: llm-env not found"
source ~/.local/bin/llm-env
echo "URL=$LLM_URL  Model=$LLM_MODEL"
```

## Step 2 -- No System Prompt

Send a question with no system prompt and observe the default behavior:

```bash
source ~/.local/bin/llm-env
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "user", "content": "What is a Pod in Kubernetes?"}
    ],
    "max_tokens": 256
  }' | jq -r '.choices[0].message.content'
```

Point out to the user: without a system prompt the model defaults to a generic assistant tone. The response is likely long and conversational.

## Step 3 -- With System Prompt

Same question, but now with a system prompt that constrains the format:

```bash
source ~/.local/bin/llm-env
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "system", "content": "You are a Kubernetes expert. Answer in one sentence, no preamble."},
      {"role": "user", "content": "What is a Pod in Kubernetes?"}
    ],
    "max_tokens": 256
  }' | jq -r '.choices[0].message.content'
```

Compare with Step 2. The system prompt compressed the response to one sentence. This is how you steer model behavior without changing the question. Smaller models benefit more from explicit system prompts than large cloud models.

## Step 4 -- Temperature

Run the same creative prompt at two temperatures to see the difference:

```bash
source ~/.local/bin/llm-env
echo "=== Temperature 0.0 (deterministic) ==="
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "system", "content": "Suggest a creative project name."},
      {"role": "user", "content": "Name a Kubernetes monitoring tool in one word."}
    ],
    "max_tokens": 32,
    "temperature": 0.0
  }' | jq -r '.choices[0].message.content'

echo ""
echo "=== Temperature 1.2 (creative) ==="
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "system", "content": "Suggest a creative project name."},
      {"role": "user", "content": "Name a Kubernetes monitoring tool in one word."}
    ],
    "max_tokens": 32,
    "temperature": 1.2
  }' | jq -r '.choices[0].message.content'
```

Temperature 0.0 gives the same answer every time you run it. Temperature 1.2 gives varied, sometimes surprising answers. Use low temperature for code and facts, higher for brainstorming.

## Verification

Run all checks:

```bash
PASS=0
TOTAL=3

# 1. llm-env helper exists
[ -f "$HOME/.local/bin/llm-env" ] && { echo "PASS: llm-env helper"; PASS=$((PASS+1)); } || echo "FAIL: llm-env helper missing"

# 2. LiteLLM responding
source ~/.local/bin/llm-env
curl -sf "$LLM_URL/health" --max-time 5 &>/dev/null && { echo "PASS: LiteLLM responding"; PASS=$((PASS+1)); } || echo "FAIL: LiteLLM not reachable"

# 3. Model query works
RESP=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"'"$LLM_MODEL"'","messages":[{"role":"user","content":"say ok"}],"max_tokens":16}' \
  --max-time 30 | jq -r '.choices[0].message.content // empty' 2>/dev/null)
[ -n "$RESP" ] && { echo "PASS: Model responding ($LLM_MODEL)"; PASS=$((PASS+1)); } || echo "FAIL: No response from model"

echo ""
echo "$PASS/$TOTAL checks passed."
```

If all 3 pass:
```
All checks passed. You can query the LLM with system prompts and temperature control.
```

If any fail, tell the user which step to re-run.

## Challenge

```
Write a system prompt that makes the model return ONLY valid YAML -- no
markdown fences, no explanation, just raw YAML.

Test it by asking for a Kubernetes ConfigMap named "app-config" with:
  APP_ENV=production
  LOG_LEVEL=info

Verify your output:
  echo 'YOUR_YAML_HERE' | python3 -c "import yaml, sys; yaml.safe_load(sys.stdin) and print('Valid YAML')"

Tell me:
  1. Your system prompt
  2. Whether the YAML passed validation on the first try
```

## Challenge Verification

Ask the user for their system prompt. Then test it:

```bash
source ~/.local/bin/llm-env
RESP=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model":"'"$LLM_MODEL"'",
    "messages":[
      {"role":"system","content":"You are a YAML generator. Output ONLY valid YAML. No markdown fences, no explanation, no extra text."},
      {"role":"user","content":"Create a Kubernetes ConfigMap named app-config with data keys APP_ENV=production and LOG_LEVEL=info"}
    ],
    "max_tokens":256,
    "temperature":0.0
  }' --max-time 30 | jq -r '.choices[0].message.content' 2>/dev/null)

echo "$RESP"
echo "---"
echo "$RESP" | python3 -c "import yaml, sys; yaml.safe_load(sys.stdin) and print('Valid YAML')" 2>&1
```

If the user's system prompt produced valid YAML, congratulate them. If not, show the baseline prompt above as a working example.

If successful, write the completion marker:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/02.done
```

Then print:
```
Module 02 complete.

You can now control model output with system prompts and temperature:
  - System prompts steer tone, format, and behavior
  - Temperature 0.0 for deterministic, higher for creative
  - Smaller models need more explicit system prompts than large cloud models

Next module: /learn-03-structured-output

Questions or feedback? Open an issue in this repo.
```

<!-- NEW -->
