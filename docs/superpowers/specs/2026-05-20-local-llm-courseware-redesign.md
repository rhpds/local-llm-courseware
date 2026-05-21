# Local LLM Courseware Redesign

Date: 2026-05-20
Status: Draft
Repository: ~/repos/local-llm-courseware

## Problem Statement

The local-llm-courseware project was designed to teach the RHDP ops team how to
run Claude Code against a local Ollama backend via a LiteLLM proxy. In practice,
this approach fails:

- **Performance:** A 30B model on an 18GB M3 Pro takes 30-40 seconds per response
  and exhausts system memory. Even a 14B model produces 29-37 second response times.
- **Quality:** Claude Code's system prompt includes complex tool definitions and a
  thinking parameter designed for Claude models. Local models cannot interpret these
  correctly, producing malformed JSON (`sequentialthinking` tool calls) instead of
  usable answers.
- **Fragility:** The LiteLLM proxy's Anthropic passthrough endpoint does not respect
  `drop_params` or `model_info` settings for the thinking parameter. Workarounds
  (env vars, config flags) are unreliable and may break with upstream updates.

These are fundamental mismatches, not configuration problems. Claude Code is
purpose-built for Claude models. Forcing it to work with local models through a
proxy produces a poor experience that no amount of tuning will fix.

## Redesign Direction

Shift from "force Claude Code to work with local models" to "use the right tool
for each scenario":

- **Claude Code + Vertex AI** for cloud inference (taught in claude-code-courseware)
- **Cursor + Ollama** for local inference (taught here)
- **Infrastructure knowledge** (LiteLLM, RamaLama, model management) as ops skills

All tools referenced are on the team's approved tools list.

## Approved Tools Referenced

| Tool | Role in Courseware |
|------|--------------------|
| Ollama | Primary local model runtime |
| Cursor | Local coding assistant IDE |
| RamaLama | Alternative container-native runtime (Red Hat) |
| Granite.code | Red Hat/IBM model used in RamaLama challenge |
| LiteLLM | API proxy teaching tool (infrastructure module) |

## Target Audience

RHDP ops team with wide-ranging hardware (16GB-64GB+ Macs). Modules must work
on 16GB minimum and scale gracefully on beefier machines.

## Hard Constraints

Carried from the existing repository:

1. Local Ollama backend only. No remote providers.
2. LiteLLM proxy binds to 127.0.0.1 only.
3. LiteLLM version >= 1.83 (PyPI 1.82.7 and 1.82.8 were compromised).
4. No external-provider scaffolding in config files.
5. Extensible by appending (configs structured for easy model addition).

## Module Structure

Four modules, ordered by dependency. Each follows the existing template:
Orientation, Preflight, Steps, Verification, Challenge, Challenge Verification.

### Module 01: Ollama Setup + Model Management (~20 min)

**Prereqs:** macOS with zsh, Homebrew

**What it teaches:**
- Install Ollama via Homebrew
- Hardware-aware model selection (auto-detect RAM, recommend appropriate model)
- Model management commands: `ollama list`, `ollama pull`, `ollama rm`, `ollama show`, `ollama run`
- Direct chat via `ollama run` for quick testing

**Hardware-aware model selection:**

| RAM | Recommended Model | Size | Rationale |
|-----|-------------------|------|-----------|
| 16GB | qwen2.5-coder:7b | ~4.5 GB | Leaves ~11GB for OS/apps. Responsive. |
| 18GB | qwen2.5-coder:7b | ~4.5 GB | 14B is too tight at 18GB (proven in testing). |
| 24GB | qwen2.5-coder:14b | ~9 GB | Good balance of quality and headroom. |
| 32GB+ | qwen2.5-coder:14b | ~9 GB | Could go larger but diminishing returns. |
| 64GB+ | qwen3-coder:30b | ~18 GB | Enough headroom for the MoE model. |

The preflight script detects RAM via `sysctl hw.memsize`, converts to GB, and
recommends the right model. The user can override.

**Challenge:** Pull a second model, compare speed vs quality tradeoffs.

**Changes from current Module 01:** All LiteLLM, proxy, and `claude-litellm`
content moves to Module 04. Model selection is hardware-aware instead of
hardcoding qwen3-coder:30b.

### Module 02: Cursor + Ollama Integration (~25 min)

**Prereqs:** Module 01 complete, Cursor installed

**This is the primary learning outcome** -- a working local coding assistant.

**Why Cursor works where Claude Code didn't:**
- Cursor is designed to work with any OpenAI-compatible API
- Ollama exposes one natively at `http://localhost:11434/v1`
- No proxy layer needed -- direct connection
- No thinking parameter issues -- Cursor manages its own prompting
- Cursor's system prompt is lighter and model-agnostic

**Steps:**

1. **Preflight** -- Verify Cursor installed, Ollama running, model available
2. **Set OLLAMA_ORIGINS** -- Add `export OLLAMA_ORIGINS="*"` to `~/.zshrc` so
   Ollama accepts requests from Cursor. Restart Ollama.
3. **Configure Cursor** -- Settings > Models > OpenAI API:
   - Base URL: `http://localhost:11434/v1`
   - API Key: `ollama` (any string -- Ollama doesn't need auth)
   - Model name: exact match from `ollama list`
   - Deselect all non-local models to avoid validation conflicts
4. **Verify connection** -- Click Verify in Cursor, open Chat panel, test message
5. **Hands-on exercises** -- Three progressively harder tasks:
   - Generate a Python function with docstring
   - Explain an unfamiliar code snippet from team repos
   - Inline edit: refactor a function using Cursor's inline edit
6. **Teach the boundaries** -- Explicit section:
   - Works: Chat, inline edits, code generation
   - Does not work: Tab autocomplete (locked to Cursor's proprietary model)
   - When to reach for Claude Code: complex multi-file changes, architecture
     decisions, tool use / MCP integrations

**Challenge:** Pick a real file from any team repo. Use Cursor Chat with the
local model to add error handling, then use inline edit to refactor. Compare
the experience to the same task in Claude Code via Vertex.

### Module 03: RamaLama as Alternative Runtime (~20 min)

**Prereqs:** Module 01 complete, Podman available

**Purpose:** Teach the Red Hat-native option for running local models. Not a
replacement for Ollama, but an alternative the team should know exists.

**Why RamaLama matters:**
- Container-native: models run in isolated containers with security defaults
  (read-only model mounts, no network access by default)
- Multiple backends: llama.cpp, MLX, or vLLM depending on hardware
- Path to production: generates Quadlet files and K8s manifests from the same
  model setup used on a laptop
- Pulls models from Hugging Face, Ollama registry, and OCI registries
- Red Hat supported

**On macOS:** RamaLama requires Podman (container-based). The team already has
Podman Desktop approved and installed.

**Steps:**

1. **Preflight** -- Verify Podman available, Module 01 complete
2. **What RamaLama is** -- Context: Ollama for dev/prototyping, RamaLama for
   container isolation and production path
3. **Install RamaLama** -- On macOS, install via `pip install ramalama` or
   `brew install ramalama` (if available). The module must verify the install
   method at runtime since availability varies by macOS version and Homebrew state.
4. **Pull and run a model** -- Same model from Module 01 through RamaLama.
   Compare: `ramalama run` for chat, `ramalama serve` for API
5. **Connect Cursor to RamaLama** -- Same as Module 02 but pointing at
   RamaLama's serve endpoint. Demonstrate that the frontend is runtime-agnostic.
6. **Teach the tradeoffs:**
   - Ollama: simpler setup, larger model library, better macOS-native experience
   - RamaLama: container isolation, security defaults, production path, Red Hat supported

**Challenge:** Serve a Granite.code model via RamaLama, connect Cursor, compare
output quality to the Qwen model from Module 01.

### Module 04: LiteLLM Proxy Concepts (~20 min)

**Prereqs:** Module 01 complete, uv installed

**Purpose:** Teach how API translation proxies work. Infrastructure/ops
knowledge, not a daily-use workflow recommendation.

**Steps:**

1. **Preflight** -- Module 01 complete, uv installed
2. **What is an API proxy** -- Context: different AI tools speak different API
   dialects (Anthropic Messages API, OpenAI Chat Completions, Ollama native).
   Proxies translate between them. Same concept as API gateways.
3. **Set up LiteLLM** -- Create config, .env (with export prefix), and proxy
   launcher. Key fixes discovered during development:
   - `.env` must use `export` for variables to reach child processes
   - Health check must handle auth (use connectivity check, not 200 assertion)
4. **Explore API translation** -- Hands-on curl exercises:
   - Hit Ollama directly at `/api/chat` (native format)
   - Hit LiteLLM at `/v1/chat/completions` (OpenAI format)
   - Hit LiteLLM at `/v1/messages` (Anthropic format)
   - Same model, same question, three API formats
5. **Understand the limitations** -- Why Claude Code + LiteLLM + Ollama fails:
   - Claude Code's system prompt is designed for Claude models
   - Thinking parameter incompatibility
   - Smaller models can't parse complex tool definitions
   - This is a mismatch problem, not a configuration problem
6. **Multi-model routing** -- Add a second model, show how LiteLLM routes
   requests to different backends based on model name

**Challenge:** Add a third model to the LiteLLM config. Write a shell script
that sends the same prompt to all three models via the proxy and compares
response time and output.

## Courseware-Level Changes

### README.md
- New positioning: "Right tool for each scenario"
- Updated prerequisites: add Cursor
- Module table reflects four-module structure
- Add "Which Tool When?" section alongside "Why Local?"

### CLAUDE.md
- Remove hardcoded model references (qwen3-coder:30b)
- Update repo description to reflect multi-tool approach
- Hard constraints unchanged

### Module Template (TEMPLATE.md)
- No structural changes

### Shell Function (claude-litellm)
- Stays in ~/.zshrc with fixes applied during this session:
  - `export` in .env file
  - Connectivity-based health check (not HTTP 200 assertion)
  - `ANTHROPIC_API_KEY` instead of `ANTHROPIC_AUTH_TOKEN`
  - `--bare` flag to skip OAuth/keychain
  - `CLAUDE_CODE_DISABLE_THINKING=1`
- Documented in Module 04 as a teaching tool, not recommended workflow

### Existing Module 01 File
- `modules/01-litellm-ollama-setup.md` gets rewritten as new Ollama-focused Module 01
- Old LiteLLM content preserved in Module 04

## Decision Log

| Decision | Rationale |
|----------|-----------|
| Cursor over Claude Code for local | Claude Code's system prompt is purpose-built for Claude. Cursor works with any OpenAI-compatible API natively. |
| Hardware-aware model selection | 18GB qwen3-coder:30b on 18GB RAM caused OOM/swap. Auto-detect prevents this. |
| Keep LiteLLM as teaching module | API proxy concepts have infrastructure value even though the Claude Code workflow doesn't work well. |
| Include RamaLama | Approved tool, Red Hat-native, teaches container-native model serving with production path. |
| Tab autocomplete limitation documented | Cursor locks Tab to its proprietary model. Honest documentation prevents frustration. |

## References

- [Cursor + Ollama setup guides](https://dev.to/0xkoji/use-local-llm-with-cursor-2h4i)
- [RamaLama GitHub](https://github.com/containers/ramalama)
- [RamaLama - Red Hat Blog](https://www.redhat.com/en/blog/run-containerized-ai-models-locally-ramalama)
- [Serve Granite with RamaLama - Red Hat Developer](https://developers.redhat.com/learning/learn:openshift-ai:how-run-ai-models-cloud-development-environments/resource/resources:how-serve-ibm-granite-ramalama-red-hat-openshift-dev-spaces)
- [LiteLLM Reasoning Content Docs](https://docs.litellm.ai/docs/reasoning_content)
- [Ollama model library](https://ollama.com/library)
