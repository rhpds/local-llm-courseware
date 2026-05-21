# Module 03 -- Structured Output for Automation

Estimated time: 10 minutes (5 minutes click-to-paste)
Prerequisites: Module 02 complete (llm-env helper created)

Use a local LLM to generate valid JSON and YAML for automation pipelines. Pipe model output directly into tooling.

## Orientation

Print this once at the start:

```
You're learning to use LLM output in scripts and pipelines.
This takes about 10 minutes.

We'll explore:
  1. JSON generation (and common failure modes)
  2. Fixing output with system prompts
  3. Piping LLM output into oc/kubectl

You'll need: LiteLLM running, jq, oc or kubectl
```

## Progress Tracking

On module start, write a progress marker:

```bash
mkdir -p ~/.claude/courseware-progress && date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/03.started
```

## Preflight

```bash
# 1. llm-env helper
[ -f "$HOME/.local/bin/llm-env" ] && echo "EXISTS: llm-env helper" || echo "MISSING: llm-env helper (run Module 02 first)"

# 2. LiteLLM reachable
source ~/.local/bin/llm-env 2>/dev/null
curl -sf "$LLM_URL/health" --max-time 5 &>/dev/null && echo "EXISTS: LiteLLM at $LLM_URL" || echo "MISSING: LiteLLM not reachable"

# 3. jq
command -v jq &>/dev/null && echo "EXISTS: jq" || echo "MISSING: jq"

# 4. oc or kubectl
if command -v oc &>/dev/null; then
  echo "EXISTS: oc ($(oc version --client 2>/dev/null | head -1))"
elif command -v kubectl &>/dev/null; then
  echo "EXISTS: kubectl ($(kubectl version --client --short 2>/dev/null))"
else
  echo "MISSING: oc or kubectl (needed for dry-run validation)"
fi
```

Print a summary. Skip any step where prerequisites are missing.

## Step 1 -- Ask for JSON (the Naive Way)

Ask the model for JSON without any format constraints:

```bash
source ~/.local/bin/llm-env
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "user", "content": "Give me a JSON object with fields: name, replicas, image for an nginx deployment."}
    ],
    "max_tokens": 256
  }' | jq -r '.choices[0].message.content'
```

Point out the common problems: markdown fences (````json```), trailing explanation, or extra text before/after the JSON. This output breaks if you pipe it into `jq` or a script.

## Step 2 -- Fix It with a System Prompt

Add a system prompt that enforces raw JSON output, then validate with jq:

```bash
source ~/.local/bin/llm-env
RESP=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "system", "content": "You are a JSON generator. Output ONLY valid JSON. No markdown fences, no explanation, no extra text. Start with { or [."},
      {"role": "user", "content": "Give me a JSON object with fields: name (string), replicas (integer), image (string) for an nginx deployment."}
    ],
    "max_tokens": 256,
    "temperature": 0.0
  }' --max-time 30 | jq -r '.choices[0].message.content' 2>/dev/null)

echo "$RESP"
echo "---"
echo "$RESP" | jq . 2>&1
```

If jq parses it without errors, the output is valid JSON. The system prompt plus temperature 0.0 makes the model reliable enough for scripting.

## Step 3 -- Pipeline: Description to Kubernetes Manifest

Chain the LLM into a real workflow: natural language in, validated manifest out:

```bash
source ~/.local/bin/llm-env
MANIFEST=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "system", "content": "You are a Kubernetes YAML generator. Output ONLY a valid Kubernetes Deployment manifest. No markdown fences, no explanation. Start directly with apiVersion."},
      {"role": "user", "content": "Create a Deployment named web-app with 2 replicas running nginx:1.25, port 80, memory limit 128Mi."}
    ],
    "max_tokens": 512,
    "temperature": 0.0
  }' --max-time 60 | jq -r '.choices[0].message.content' 2>/dev/null)

echo "$MANIFEST"
echo "---"
echo "$MANIFEST" | python3 -c "import yaml, sys; yaml.safe_load(sys.stdin) and print('Valid YAML')" 2>&1
```

If you have `oc` or `kubectl` access, validate against the API:

```bash
source ~/.local/bin/llm-env
MANIFEST=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "system", "content": "You are a Kubernetes YAML generator. Output ONLY a valid Kubernetes Deployment manifest. No markdown fences, no explanation. Start directly with apiVersion."},
      {"role": "user", "content": "Create a Deployment named web-app with 2 replicas running nginx:1.25, port 80, memory limit 128Mi."}
    ],
    "max_tokens": 512,
    "temperature": 0.0
  }' --max-time 60 | jq -r '.choices[0].message.content' 2>/dev/null)

echo "$MANIFEST" | oc apply --dry-run=client -f - 2>&1 || echo "$MANIFEST" | kubectl apply --dry-run=client -f - 2>&1
```

This is the pattern: LLM generates structured output, tooling validates it. No human edits in between.

## Verification

```bash
PASS=0
TOTAL=2

source ~/.local/bin/llm-env

# 1. Can generate valid JSON
JSON=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"'"$LLM_MODEL"'","messages":[{"role":"system","content":"Output ONLY valid JSON. No fences."},{"role":"user","content":"JSON object with key greeting and value hello"}],"max_tokens":64,"temperature":0.0}' \
  --max-time 30 | jq -r '.choices[0].message.content' 2>/dev/null)
echo "$JSON" | jq . &>/dev/null && { echo "PASS: Valid JSON generation"; PASS=$((PASS+1)); } || echo "FAIL: JSON generation produced invalid output"

# 2. Can generate valid YAML
YAML=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"'"$LLM_MODEL"'","messages":[{"role":"system","content":"Output ONLY valid YAML. No fences."},{"role":"user","content":"YAML with key greeting and value hello"}],"max_tokens":64,"temperature":0.0}' \
  --max-time 30 | jq -r '.choices[0].message.content' 2>/dev/null)
echo "$YAML" | python3 -c "import yaml, sys; yaml.safe_load(sys.stdin)" &>/dev/null && { echo "PASS: Valid YAML generation"; PASS=$((PASS+1)); } || echo "FAIL: YAML generation produced invalid output"

echo ""
echo "$PASS/$TOTAL checks passed."
```

If all pass:
```
All checks passed. You can generate structured output and validate it programmatically.
```

## Challenge

```
Build a one-liner that takes an app description as an argument and
produces a validated Kubernetes manifest.

Example usage:
  ./gen-manifest.sh "redis with 1 replica, port 6379, 256Mi memory"

The script should:
  1. Send the description to the LLM with a YAML-enforcing system prompt
  2. Validate the output with oc/kubectl dry-run or python yaml parser
  3. Print the manifest if valid, or an error if not

Tell me:
  1. Your script (or one-liner)
  2. Whether it produced a valid manifest
```

## Challenge Verification

The user should show a working script or one-liner. Verify by asking them to run it with a test description. A minimal working version:

```bash
source ~/.local/bin/llm-env
DESC="${1:-nginx with 1 replica on port 80}"
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model":"'"$LLM_MODEL"'",
    "messages":[
      {"role":"system","content":"Output ONLY a valid Kubernetes Deployment YAML manifest. No markdown, no explanation."},
      {"role":"user","content":"'"$DESC"'"}
    ],"max_tokens":512,"temperature":0.0}' \
  --max-time 60 | jq -r '.choices[0].message.content' 2>/dev/null \
  | tee /dev/stderr | python3 -c "import yaml,sys; yaml.safe_load(sys.stdin) and print('Valid YAML')" 2>&1
```

If the user's approach produces a valid manifest, congratulate them. The key insight is: system prompt + temperature 0.0 + validation = reliable pipeline.

If successful, write the completion marker:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/03.done
```

Then print:
```
Module 03 complete.

You can now use LLM output in automation pipelines:
  - System prompts enforce output format (JSON, YAML, plain text)
  - Temperature 0.0 makes output deterministic and parseable
  - Always validate LLM output before trusting it

Next module: /learn-04-rag-chat-with-your-docs

Questions or feedback? Open an issue in this repo.
```

<!-- NEW -->
