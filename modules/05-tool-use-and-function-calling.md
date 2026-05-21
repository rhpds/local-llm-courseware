# Module 05 -- Tool Use and Function Calling

Estimated time: 10 minutes (5 minutes click-to-paste)
Prerequisites: Module 02 complete (llm-env helper created)

Teach the model to call functions by name, then execute them and feed the results back. This is how agents work.

## Orientation

Print this once at the start:

```
You're learning tool use and function calling.
This takes about 10 minutes.

We'll explore:
  1. Defining tools the model can call
  2. Executing tool calls and returning results
  3. Giving the model multiple tools to choose from

You'll need: LiteLLM running, jq
```

## Progress Tracking

On module start, write a progress marker:

```bash
mkdir -p ~/.claude/courseware-progress && date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/05.started
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
```

## Step 1 -- Define a Tool

Send a request with a tool definition. The model does not answer directly -- it returns a tool call instead:

```bash
source ~/.local/bin/llm-env
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "user", "content": "What time is it right now?"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_current_time",
          "description": "Get the current date and time",
          "parameters": {
            "type": "object",
            "properties": {
              "timezone": {
                "type": "string",
                "description": "Timezone like UTC, US/Eastern, Europe/London"
              }
            },
            "required": ["timezone"]
          }
        }
      }
    ],
    "max_tokens": 256
  }' | jq '.choices[0].message'
```

Point out: instead of guessing the time, the model returned a `tool_calls` array with `function.name` and `function.arguments`. It decided WHEN to call the tool and WHAT arguments to pass. You decide how to execute it.

## Step 2 -- Execute and Return

Complete the loop: execute the tool call, send the result back, get a natural-language answer:

```bash
source ~/.local/bin/llm-env

FIRST_RESPONSE=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [
      {"role": "user", "content": "What is the current date and time?"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_current_time",
          "description": "Get the current date and time",
          "parameters": {
            "type": "object",
            "properties": {
              "timezone": {"type": "string", "description": "Timezone like UTC"}
            },
            "required": ["timezone"]
          }
        }
      }
    ],
    "max_tokens": 256
  }')

TOOL_CALL_ID=$(echo "$FIRST_RESPONSE" | jq -r '.choices[0].message.tool_calls[0].id // empty')
ASSISTANT_MSG=$(echo "$FIRST_RESPONSE" | jq '.choices[0].message')

if [ -n "$TOOL_CALL_ID" ]; then
  CURRENT_TIME=$(date -u '+%Y-%m-%d %H:%M:%S UTC')
  echo "Tool called. Executing: get_current_time -> $CURRENT_TIME"
  echo "---"

  curl -s "$LLM_URL/v1/chat/completions" \
    -H "Authorization: Bearer $LLM_KEY" \
    -H "Content-Type: application/json" \
    -d '{
      "model": "'"$LLM_MODEL"'",
      "messages": [
        {"role": "user", "content": "What is the current date and time?"},
        '"$ASSISTANT_MSG"',
        {"role": "tool", "tool_call_id": "'"$TOOL_CALL_ID"'", "content": "'"$CURRENT_TIME"'"}
      ],
      "max_tokens": 256
    }' | jq -r '.choices[0].message.content'
else
  echo "Model did not make a tool call. Response:"
  echo "$FIRST_RESPONSE" | jq -r '.choices[0].message.content'
fi
```

The model now answers with the real time. The flow is: user asks -> model calls tool -> you execute -> model summarizes. This is the agent loop.

## Step 3 -- Multiple Tools

Give the model several tools and let it pick the right one:

```bash
source ~/.local/bin/llm-env

TOOLS='[
  {
    "type": "function",
    "function": {
      "name": "get_current_time",
      "description": "Get the current date and time in a timezone",
      "parameters": {
        "type": "object",
        "properties": {
          "timezone": {"type": "string", "description": "Timezone like UTC"}
        },
        "required": ["timezone"]
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "get_disk_usage",
      "description": "Get disk usage for a filesystem path",
      "parameters": {
        "type": "object",
        "properties": {
          "path": {"type": "string", "description": "Filesystem path to check"}
        },
        "required": ["path"]
      }
    }
  },
  {
    "type": "function",
    "function": {
      "name": "count_files",
      "description": "Count files in a directory",
      "parameters": {
        "type": "object",
        "properties": {
          "directory": {"type": "string", "description": "Directory path"},
          "extension": {"type": "string", "description": "File extension filter like .txt or .yaml"}
        },
        "required": ["directory"]
      }
    }
  }
]'

echo "=== Question about time ==="
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [{"role": "user", "content": "What time is it in UTC?"}],
    "tools": '"$TOOLS"',
    "max_tokens": 256
  }' | jq '.choices[0].message.tool_calls[0].function'

echo ""
echo "=== Question about disk ==="
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [{"role": "user", "content": "How much disk space is used on /home?"}],
    "tools": '"$TOOLS"',
    "max_tokens": 256
  }' | jq '.choices[0].message.tool_calls[0].function'

echo ""
echo "=== Question about files ==="
curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$LLM_MODEL"'",
    "messages": [{"role": "user", "content": "How many .yaml files are in /etc/kubernetes?"}],
    "tools": '"$TOOLS"',
    "max_tokens": 256
  }' | jq '.choices[0].message.tool_calls[0].function'
```

The model picks `get_current_time`, `get_disk_usage`, and `count_files` respectively. It reads the tool descriptions and routes the question to the right function. This is how Claude Code, ChatGPT plugins, and MCP all work under the hood.

## Verification

```bash
PASS=0
TOTAL=2

source ~/.local/bin/llm-env

# 1. Model returns tool_calls when given a tool
RESP=$(curl -s "$LLM_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"'"$LLM_MODEL"'","messages":[{"role":"user","content":"What time is it?"}],"tools":[{"type":"function","function":{"name":"get_time","description":"Get current time","parameters":{"type":"object","properties":{"tz":{"type":"string"}},"required":["tz"]}}}],"max_tokens":256}' \
  --max-time 30)
TOOL_NAME=$(echo "$RESP" | jq -r '.choices[0].message.tool_calls[0].function.name // empty' 2>/dev/null)
[ "$TOOL_NAME" = "get_time" ] && { echo "PASS: Model returned tool call ($TOOL_NAME)"; PASS=$((PASS+1)); } || echo "FAIL: Expected tool call, got: $(echo "$RESP" | jq -r '.choices[0].message.content // .choices[0].message' 2>/dev/null)"

# 2. Full tool-call loop completes
if [ -n "$TOOL_NAME" ]; then
  TOOL_ID=$(echo "$RESP" | jq -r '.choices[0].message.tool_calls[0].id')
  ASST=$(echo "$RESP" | jq '.choices[0].message')
  FINAL=$(curl -s "$LLM_URL/v1/chat/completions" \
    -H "Authorization: Bearer $LLM_KEY" \
    -H "Content-Type: application/json" \
    -d '{"model":"'"$LLM_MODEL"'","messages":[{"role":"user","content":"What time is it?"},'"$ASST"',{"role":"tool","tool_call_id":"'"$TOOL_ID"'","content":"2025-01-15 14:30:00 UTC"}],"max_tokens":256}' \
    --max-time 30 | jq -r '.choices[0].message.content // empty' 2>/dev/null)
  [ -n "$FINAL" ] && { echo "PASS: Tool loop completed with response"; PASS=$((PASS+1)); } || echo "FAIL: No response after tool result"
else
  echo "SKIP: Cannot test tool loop (tool call failed)"
fi

echo ""
echo "$PASS/$TOTAL checks passed."
```

## Challenge

```
Define a new tool called "get_weather" that takes a "city" parameter.

1. Send a question like "What's the weather in Tokyo?"
2. The model should call your get_weather tool
3. Return a fake weather result (e.g., "22C, partly cloudy")
4. Get the model's final natural-language response

Tell me:
  1. Your tool definition
  2. The model's final response after receiving the weather data
```

## Challenge Verification

The user should show:
1. A tool definition with `name: get_weather` and a `city` parameter
2. The model calling `get_weather` with a city argument
3. A final response that incorporates the fake weather data

If successful, write the completion marker:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/05.done
```

Then print:
```
Module 05 complete.

You now understand tool use and function calling:
  - Define tools with name, description, and parameters
  - The model decides when to call and what arguments to pass
  - You execute the function and return results
  - This is how agents, plugins, and MCP work under the hood

Next module: /learn-06-mcp-server

Questions or feedback? Open an issue in this repo.
```

<!-- NEW -->
