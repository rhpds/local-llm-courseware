# Local LLM Courseware

Hands-on learning modules for running Claude Code against a local Ollama backend via LiteLLM proxy. Fully offline, AUP-safe, no API keys required.

This course runs parallel to the [Claude Code Courseware](https://github.com/rhpds/claude-code-courseware) (Vertex AI backend). Where that course routes calls through Google Cloud, this one keeps everything on your machine.

## Getting Started

1. Clone this repo:
   ```bash
   git clone git@github.com:rhpds/local-llm-courseware.git
   cd local-llm-courseware
   ```

2. Launch Claude Code (use your work or personal profile):
   ```bash
   claude-work
   ```

3. See what's available:
   ```
   /courseware
   ```

4. Start a module:
   ```
   /learn-01-litellm-ollama-setup
   ```

### Check Prerequisites

Run `/preflight` to verify your environment before starting modules.

## Modules

### Setup & Foundation

| # | Title | Time | Description |
|---|-------|------|-------------|
| 01 | LiteLLM + Ollama Setup | ~20 min | Install Ollama, configure LiteLLM proxy, and create the `claude-litellm` profile for local-only inference |

## Prerequisites

- macOS with zsh (Linux support planned)
- Homebrew installed
- ~20 GB free disk space (for model weights)
- Existing `claude-work` and/or `claude-personal` profiles (recommended, not required)

## How Modules Work

Every module follows the same pattern:

1. **Orientation** -- what you'll learn
2. **Preflight** -- checks what's already set up, skips what's done
3. **Steps** -- guided walkthrough with verification at each step
4. **Verification** -- all-green final check
5. **Challenge** -- hands-on task to solidify the concepts

## Why Local?

- **AUP-safe**: no data leaves your machine. Experiment with sensitive inputs, proprietary code, or internal docs without compliance concerns.
- **Offline**: works on planes, trains, and air-gapped networks.
- **Cost-free inference**: after the initial model download, every query is free.
- **Experimentation**: try different models, parameters, and prompts without burning tokens.

The tradeoff: local models are smaller and less capable than cloud-hosted Claude. This course teaches you when local inference is the right tool and when to reach for `claude-work` or `claude-personal` instead.

## Authoring New Modules

1. Create `modules/NN-topic.md` following `modules/TEMPLATE.md`
2. Create `.claude/commands/learn-NN-topic.md` as a thin dispatcher
3. Add the module to the catalog in `.claude/commands/courseware.md`
4. Update the tables in this README
