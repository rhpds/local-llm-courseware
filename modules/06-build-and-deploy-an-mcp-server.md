# Module 06 -- Build and Deploy an MCP Server

Estimated time: 15 minutes (10 minutes click-to-paste)
Prerequisites: Module 05 complete (understands tool use), Python 3.10+, uv

Build a Model Context Protocol server that exposes tools backed by your local LLM. Connect it to Claude Code so it can call your tools natively.

## Orientation

Print this once at the start:

```
You're building an MCP server.
This takes about 15 minutes.

We'll build:
  1. A minimal MCP server in Python (~40 lines)
  2. Connect it to Claude Code
  3. Test it with a live tool call

You'll need: LiteLLM running, Python 3.10+, uv
```

## Progress Tracking

On module start, write a progress marker:

```bash
mkdir -p ~/.claude/courseware-progress && date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/06.started
```

## Preflight

```bash
# 1. llm-env helper
[ -f "$HOME/.local/bin/llm-env" ] && echo "EXISTS: llm-env helper" || echo "MISSING: llm-env helper (run Module 02 first)"

# 2. LiteLLM reachable
source ~/.local/bin/llm-env 2>/dev/null
curl -sf "$LLM_URL/health" --max-time 5 &>/dev/null && echo "EXISTS: LiteLLM at $LLM_URL" || echo "MISSING: LiteLLM not reachable"

# 3. Python 3.10+
if command -v python3 &>/dev/null; then
  PY_VER=$(python3 -c "import sys; print(f'{sys.version_info.major}.{sys.version_info.minor}')" 2>/dev/null)
  python3 -c "import sys; assert sys.version_info >= (3,10)" 2>/dev/null && echo "EXISTS: Python $PY_VER" || echo "MISSING: Python 3.10+ required (found $PY_VER)"
else
  echo "MISSING: python3"
fi

# 4. uv
command -v uv &>/dev/null && echo "EXISTS: uv ($(uv --version 2>/dev/null))" || echo "MISSING: uv (install: brew install uv)"

# 5. MCP project directory
[ -d "$HOME/mcp-llm-tools" ] && echo "EXISTS: ~/mcp-llm-tools project" || echo "MISSING: ~/mcp-llm-tools (will create in Step 1)"
```

## Step 1 -- Scaffold the MCP Server

Create a project directory with a single Python file that uses inline script metadata so `uv run` handles dependencies automatically:

```
Creating the MCP server project.

  ! mkdir -p ~/mcp-llm-tools && cat > ~/mcp-llm-tools/server.py << 'MCPSERVER'
# /// script
# requires-python = ">=3.10"
# dependencies = ["mcp[cli]>=1.0", "httpx"]
# ///
"""MCP server that exposes local LLM tools."""
import os
import httpx
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("local-llm-tools")

LLM_URL = os.environ.get("LLM_URL", "http://127.0.0.1:4000")
LLM_KEY = os.environ.get("LLM_KEY", "")
LLM_MODEL = os.environ.get("LLM_MODEL", "qwen3-coder")


def _query(system: str, prompt: str, max_tokens: int = 512) -> str:
    resp = httpx.post(
        f"{LLM_URL}/v1/chat/completions",
        headers={"Authorization": f"Bearer {LLM_KEY}"},
        json={
            "model": LLM_MODEL,
            "messages": [
                {"role": "system", "content": system},
                {"role": "user", "content": prompt},
            ],
            "max_tokens": max_tokens,
            "temperature": 0.0,
        },
        timeout=60.0,
    )
    resp.raise_for_status()
    return resp.json()["choices"][0]["message"]["content"]


@mcp.tool()
def ask_local_llm(question: str) -> str:
    """Ask the local LLM a question. Useful for getting a second opinion
    from a model running entirely on-premises with no data leaving the
    cluster."""
    return _query("Answer concisely.", question)


@mcp.tool()
def generate_yaml(description: str) -> str:
    """Generate a Kubernetes YAML manifest from a natural-language
    description. Returns raw YAML ready for oc/kubectl apply."""
    return _query(
        "You are a Kubernetes YAML generator. Output ONLY valid YAML. "
        "No markdown fences, no explanation. Start with apiVersion.",
        description,
        max_tokens=1024,
    )


if __name__ == "__main__":
    mcp.run()
MCPSERVER
```

Verify:

```bash
[ -f "$HOME/mcp-llm-tools/server.py" ] && echo "PASS: server.py created" || echo "FAIL: server.py not found"
python3 -c "import ast; ast.parse(open('$HOME/mcp-llm-tools/server.py').read()); print('PASS: Python syntax valid')"
```

## Step 2 -- Test the Server Standalone

Before connecting to Claude Code, verify the server starts and registers its tools:

```bash
source ~/.local/bin/llm-env
cd ~/mcp-llm-tools
export LLM_URL LLM_KEY LLM_MODEL
timeout 15 uv run server.py --help 2>&1 || echo "(timeout is expected -- the server runs until stopped)"
```

List the registered tools:

```bash
source ~/.local/bin/llm-env
cd ~/mcp-llm-tools
export LLM_URL LLM_KEY LLM_MODEL
uv run mcp list-tools server.py 2>/dev/null || echo "Tools: ask_local_llm, generate_yaml (verify by connecting to Claude Code)"
```

If `mcp list-tools` is not available in your version, that is fine -- you will verify the tools via Claude Code in the next step.

## Step 3 -- Connect to Claude Code

Register the MCP server with Claude Code. This uses the `claude mcp add` command:

```
Connect the MCP server to Claude Code:

  ! source ~/.local/bin/llm-env && claude mcp add local-llm-tools \
    -e LLM_URL="$LLM_URL" \
    -e LLM_KEY="$LLM_KEY" \
    -e LLM_MODEL="$LLM_MODEL" \
    -- uv run ~/mcp-llm-tools/server.py
```

Verify the server is registered:

```bash
claude mcp list 2>/dev/null | grep -q "local-llm-tools" && echo "PASS: MCP server registered" || echo "FAIL: MCP server not found in claude mcp list"
```

Tell the user:

```
Your MCP server is now registered. Next time you start Claude Code,
it will have two new tools available:

  ask_local_llm     -- query the local model
  generate_yaml     -- create K8s manifests from descriptions

Try it: start a new Claude Code session and ask it to
"use the local LLM to explain what a DaemonSet is" or
"generate a YAML manifest for a redis deployment."

Claude Code will call your MCP server, which calls your local LLM.
No data leaves your machine.
```

## Verification

```bash
PASS=0
TOTAL=3

# 1. server.py exists and is valid Python
python3 -c "import ast; ast.parse(open('$HOME/mcp-llm-tools/server.py').read())" 2>/dev/null \
  && { echo "PASS: server.py is valid Python"; PASS=$((PASS+1)); } \
  || echo "FAIL: server.py missing or has syntax errors"

# 2. Server has both tools defined
grep -q "def ask_local_llm" "$HOME/mcp-llm-tools/server.py" 2>/dev/null \
  && grep -q "def generate_yaml" "$HOME/mcp-llm-tools/server.py" 2>/dev/null \
  && { echo "PASS: Both tools defined (ask_local_llm, generate_yaml)"; PASS=$((PASS+1)); } \
  || echo "FAIL: Missing tool definitions"

# 3. MCP server registered in Claude Code
claude mcp list 2>/dev/null | grep -q "local-llm-tools" \
  && { echo "PASS: MCP server registered with Claude Code"; PASS=$((PASS+1)); } \
  || echo "FAIL: MCP server not registered (run: claude mcp add ...)"

echo ""
echo "$PASS/$TOTAL checks passed."
```

If all 3 pass:
```
All checks passed. Your MCP server is ready.
```

## Challenge

```
Add a third tool to your MCP server. Ideas:

  summarize_text(text: str) -> str
    Summarize a long block of text using the local LLM.

  explain_error(error_message: str) -> str
    Explain an error message and suggest fixes.

  review_yaml(yaml_content: str) -> str
    Review a YAML manifest for common mistakes.

Steps:
  1. Add the tool function to server.py
  2. Re-register with: claude mcp remove local-llm-tools
     then re-run the claude mcp add command from Step 3
  3. Test it in a new Claude Code session

Tell me:
  1. Which tool you added
  2. The output when Claude Code called it
```

## Challenge Verification

Check that the user added a third tool:

```bash
TOOL_COUNT=$(grep -c '@mcp.tool()' "$HOME/mcp-llm-tools/server.py" 2>/dev/null)
[ "$TOOL_COUNT" -ge 3 ] && echo "PASS: $TOOL_COUNT tools defined" || echo "FAIL: Expected 3+ tools, found $TOOL_COUNT"
python3 -c "import ast; ast.parse(open('$HOME/mcp-llm-tools/server.py').read())" 2>/dev/null \
  && echo "PASS: Python syntax valid" || echo "FAIL: Syntax error in server.py"
```

If successful, write the completion marker:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/06.done
```

Then print:
```
Module 06 complete.

You built an MCP server backed by a local LLM:
  - MCP is how Claude Code discovers and calls external tools
  - Your server exposes local LLM capabilities as tools
  - Any MCP client (Claude Code, Cursor, Zed) can use your server
  - All inference stays local -- no data leaves your machine

This is the full stack: local model -> LiteLLM proxy -> MCP server -> Claude Code.

Questions or feedback? Open an issue in this repo.
```

<!-- NEW -->
