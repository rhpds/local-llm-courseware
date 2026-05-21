# Module 04 -- RAG: Chat with Your Docs

Estimated time: 15 minutes (8 minutes click-to-paste)
Prerequisites: Module 02 complete (llm-env helper created)

Feed the model context it was never trained on. Build a retrieval pipeline that grounds answers in your own documents.

## Orientation

Print this once at the start:

```
You're learning retrieval-augmented generation (RAG).
This takes about 15 minutes.

We'll explore:
  1. What happens when you ask about unknown topics (hallucination)
  2. How injecting context fixes it
  3. A scripted retrieval pipeline

You'll need: LiteLLM running, jq
```

## Progress Tracking

On module start, write a progress marker:

```bash
mkdir -p ~/.claude/courseware-progress && date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/04.started
```

## Preflight

```bash
# 1. llm-env helper
[ -f "$HOME/.local/bin/llm-env" ] && echo "EXISTS: llm-env helper" || echo "MISSING: llm-env helper (run Module 02 first)"

# 2. LiteLLM reachable
source ~/.local/bin/llm-env 2>/dev/null
curl -sf "$LLM_URL/health" --max-time 5 &>/dev/null && echo "EXISTS: LiteLLM at $LLM_URL" || echo "MISSING: LiteLLM not reachable"

# 3. Sample docs
[ -d "$HOME/rag-docs" ] && echo "EXISTS: ~/rag-docs directory" || echo "MISSING: ~/rag-docs (will create in Step 1)"

# 4. jq
command -v jq &>/dev/null && echo "EXISTS: jq" || echo "MISSING: jq"
```

## Step 1 -- Create Sample Documents

Skip if `~/rag-docs` already exists with content.

These are fictional internal documents the model has never seen. This lets you clearly distinguish grounded answers from hallucination.

```
Creating sample docs for RAG testing.

  ! mkdir -p ~/rag-docs && cat > ~/rag-docs/project-phoenix.txt << 'EOF'
Project Phoenix - Internal Platform Migration

Project Phoenix is an internal initiative to migrate all legacy
Jenkins pipelines to Tekton on OpenShift. The project started in
Q3 2025 and targets completion by Q2 2026. Lead engineer is Priya
Sharma. The migration covers 847 pipelines across 12 teams.

Key decision: all new pipelines must use ClusterTasks from the
shared catalog. No team-specific Task definitions without
architecture review approval.
EOF

  ! cat > ~/rag-docs/oncall-runbook.txt << 'EOF'
Oncall Runbook - Escalation Procedures

Severity 1 (service down): Page the primary oncall via PagerDuty.
If no response in 15 minutes, page the secondary. Escalation
manager is Dana Kim (dana.kim@example.com).

Severity 2 (degraded): Create a Jira ticket in PLATFORM project,
priority Critical. Notify #ops-alerts Slack channel. Do NOT page
unless degradation exceeds 30 minutes.

Severity 3 (cosmetic/minor): Create Jira ticket, priority Normal.
No paging, no Slack alert required.
EOF

  ! cat > ~/rag-docs/api-standards.txt << 'EOF'
API Standards - Internal Guidelines v2.1

All internal APIs must:
  - Use OpenAPI 3.1 spec committed to the repo root
  - Return JSON with snake_case field names
  - Include X-Request-ID header in all responses
  - Rate limit at 100 req/s per client by default
  - Use /healthz for health checks, /readyz for readiness

Authentication: mTLS between services, OAuth2 for external clients.
JWT tokens issued by the internal Keycloak instance (auth.internal).
EOF
```

Verify:

```bash
ls -la ~/rag-docs/
wc -l ~/rag-docs/*.txt
```

## Step 2 -- Ask Without Context (Hallucination)

Ask about Project Phoenix without providing any context:

```bash
source ~/.local/bin/llm-env
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "user", "content": "Who is the lead engineer on Project Phoenix and when does it target completion?"}
    ],
    "max_tokens": 256
  }' | jq -r '.choices[0].message.content'
```

Point out: the model either hallucinated (made up a name and date) or refused to answer. It has no way to know about internal projects. This is the problem RAG solves.

## Step 3 -- Manual Context Injection

Paste the relevant document into the prompt as context:

```bash
source ~/.local/bin/llm-env
CONTEXT=$(cat ~/rag-docs/project-phoenix.txt)
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "system", "content": "Answer based ONLY on the provided context. If the context does not contain the answer, say so."},
      {"role": "user", "content": "Context:\n'"$(echo "$CONTEXT" | sed 's/"/\\"/g; s/$/\\n/' | tr -d '\n')"'\n\nQuestion: Who is the lead engineer on Project Phoenix and when does it target completion?"}
    ],
    "max_tokens": 256,
    "temperature": 0.0
  }' --max-time 30 | jq -r '.choices[0].message.content'
```

Now the model answers correctly: Priya Sharma, Q2 2026. The context grounded the response. This is the core of RAG: retrieve relevant text, inject it, get grounded answers.

## Step 4 -- Scripted Retrieval Pipeline

Automate the retrieve-and-ask pattern with grep-based search:

```bash
source ~/.local/bin/llm-env

QUESTION="What is the escalation procedure for a severity 1 incident?"
DOCS_DIR="$HOME/rag-docs"

CONTEXT=$(grep -rl "$(echo "$QUESTION" | tr ' ' '\n' | sort -u | grep -E '.{4,}' | head -5 | tr '\n' '|' | sed 's/|$//')" "$DOCS_DIR" 2>/dev/null | head -3 | while read -r f; do
  echo "--- $(basename "$f") ---"
  cat "$f"
  echo ""
done)

if [ -z "$CONTEXT" ]; then
  echo "No relevant documents found."
else
  echo "Retrieved context from: $(grep -rl "$(echo "$QUESTION" | tr ' ' '\n' | sort -u | grep -E '.{4,}' | head -5 | tr '\n' '|' | sed 's/|$//')" "$DOCS_DIR" 2>/dev/null | head -3 | xargs -I{} basename {} | tr '\n' ', ')"
  echo "---"
  ESCAPED_CTX=$(echo "$CONTEXT" | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))" 2>/dev/null)
  ESCAPED_Q=$(echo "$QUESTION" | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))" 2>/dev/null)
  curl -s "$LLM_URL/v1/chat/completions" \
    -H "Authorization: Bearer $LLM_KEY" \
    -H "Content-Type: application/json" \
    -d '{
      "model": "'"$LLM_MODEL"'",
      "messages": [
        {"role": "system", "content": "Answer based ONLY on the provided context. If the context does not contain the answer, say so."},
        {"role": "user", "content": "Context:\n'"$ESCAPED_CTX"'\n\nQuestion: '"$ESCAPED_Q"'"}
      ],
      "max_tokens": 256,
      "temperature": 0.0
    }' --max-time 30 | jq -r '.choices[0].message.content'
fi
```

This is RAG at its simplest: grep finds relevant files, their content becomes the prompt context, the model answers from that context. Production RAG uses vector embeddings instead of grep, but the pattern is identical.

## Verification

```bash
PASS=0
TOTAL=2

# 1. Sample docs exist
DOC_COUNT=$(ls ~/rag-docs/*.txt 2>/dev/null | wc -l | tr -d ' ')
[ "$DOC_COUNT" -ge 3 ] && { echo "PASS: $DOC_COUNT docs in ~/rag-docs"; PASS=$((PASS+1)); } || echo "FAIL: Expected 3+ docs in ~/rag-docs"

# 2. Context-grounded query works
source ~/.local/bin/llm-env
CONTEXT=$(cat ~/rag-docs/project-phoenix.txt)
RESP=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model":"'"$LLM_MODEL"'",
    "messages":[
      {"role":"system","content":"Answer based ONLY on the context."},
      {"role":"user","content":"Context: '"$(echo "$CONTEXT" | sed 's/"/\\"/g' | tr '\n' ' ')"' Question: Who leads Project Phoenix?"}
    ],"max_tokens":64,"temperature":0.0}' \
  --max-time 30 | jq -r '.choices[0].message.content // empty' 2>/dev/null)
echo "$RESP" | grep -qi "priya" && { echo "PASS: Grounded answer mentions Priya Sharma"; PASS=$((PASS+1)); } || echo "FAIL: Expected grounded answer about Priya Sharma"

echo ""
echo "$PASS/$TOTAL checks passed."
```

## Challenge

```
Add a fourth document to ~/rag-docs about a topic of your choice
(a fictional project, process, or policy).

Then ask the retrieval pipeline a question that requires info from
your new document.

Tell me:
  1. What document you added
  2. Your question
  3. Whether the pipeline retrieved the right document and gave
     a grounded answer
```

## Challenge Verification

Ask the user to show their new document and the pipeline output. Verify:

1. The new file exists in `~/rag-docs/`
2. The pipeline retrieved the correct document
3. The answer references content from the new document, not hallucination

```bash
DOC_COUNT=$(ls ~/rag-docs/*.txt 2>/dev/null | wc -l | tr -d ' ')
[ "$DOC_COUNT" -ge 4 ] && echo "PASS: $DOC_COUNT docs in ~/rag-docs" || echo "FAIL: Expected 4+ docs"
```

If successful, write the completion marker:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/04.done
```

Then print:
```
Module 04 complete.

You now understand retrieval-augmented generation:
  - Models hallucinate on topics outside their training data
  - Injecting context into the prompt grounds the answer
  - The retrieve-then-ask pattern works at any scale
  - Production RAG uses vector embeddings; the pattern is the same

Next module: /learn-05-tool-use

Questions or feedback? Open an issue in this repo.
```

<!-- NEW -->
