# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## What This Repo Does

Interactive learning modules for running Claude Code against a local Ollama backend via LiteLLM proxy. Each module teaches a specific aspect of local-LLM workflows through a guided, hands-on walkthrough with verification and a challenge.

This course is local-only by design. No module should reference, scaffold, document, or hint at OpenAI, public Gemini, Anthropic API passthrough, Ollama Cloud, or any other remote provider. The value proposition is fully offline, AUP-safe inference.

## Repository Structure

```
local-llm-courseware/
  .claude/commands/          # skill dispatchers and menu
    courseware.md             # /courseware -- module catalog
    learn-NN-topic.md        # dispatchers that load modules/NN-topic.md
  modules/                   # full module content (the courseware)
    NN-topic.md              # one file per module
    TEMPLATE.md              # module skeleton
  scripts/                   # setup scripts
  docs/                      # documentation
```

## How Modules Work

Each module in `modules/` follows a fixed structure:
1. **Orientation** -- what you'll learn
2. **Preflight** -- audit current state, skip what's already done
3. **Steps** -- guided walkthrough with verification at each step
4. **Verification** -- all-green final check
5. **Challenge** -- hands-on task
6. **Challenge Verification** -- confirm the result

Dispatchers in `.claude/commands/` are thin (under 30 lines) and load the corresponding module file.

## Authoring New Modules

1. Create `modules/NN-topic.md` following `modules/TEMPLATE.md`
2. Create `.claude/commands/learn-NN-topic.md` as a dispatcher
3. Add the module to the catalog in `.claude/commands/courseware.md`

## Key Conventions

- Preflight checks use EXISTS/MISSING pattern (audit first, skip what's green)
- Read-only checks run automatically; system-modifying actions use `!` prefix
- No conventional-commit prefixes in commit messages or PR titles
- No emojis in any output

## Hard Constraints

These apply to every module, every commit, every file in this repo:

1. **Local Ollama backend ONLY.** Never reference, scaffold, document, or hint at OpenAI, public Gemini, Anthropic API passthrough, Ollama Cloud, or any other remote provider.
2. **LiteLLM proxy binds to 127.0.0.1 only** -- never 0.0.0.0.
3. **LiteLLM version >= 1.83** -- PyPI versions 1.82.7 and 1.82.8 were compromised. Always pin above them.
4. **No external-provider scaffolding.** Config files must not contain entries for non-Ollama providers, even commented out.
5. **Extensible by appending.** Structure configs so a future module can add another local model by appending one `model_list` entry.

## Versioning

This project uses semver tags on the `main` branch.

| Bump  | When |
|-------|------|
| Major | Breaking changes: module restructuring, incompatible command renames |
| Minor | New module added, new command added |
| Patch | Fixes or edits to existing modules, catalog, commands, or docs |

Tag format: `vX.Y.Z` (e.g. `v1.0.0`). Create annotated tags:
```
git tag -a vX.Y.Z -m "description of release"
```
