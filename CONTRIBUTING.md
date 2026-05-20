# Contributing to Local LLM Courseware

## Adding a Module

1. Write your module file at `modules/NN-topic.md` following the structure in `modules/TEMPLATE.md`.
2. Run `/build-module` in Claude Code -- it auto-generates the dispatcher, updates the catalog, and updates the README.
3. Open a pull request.

If you prefer to create files manually instead of using `/build-module`:

1. Create `modules/NN-topic.md` using `modules/TEMPLATE.md` as a guide.
2. Create `.claude/commands/learn-NN-topic.md` as a phased dispatcher (see any existing dispatcher for the template).
3. Add the module to `.claude/commands/courseware.md` in the correct section.
4. Add the module to the README table.

## Module Structure

Every module must contain these sections:

| Section | Purpose |
|---------|---------|
| **Orientation** | What the module teaches, what the user will have when done |
| **Preflight** | Audit current state with EXISTS/MISSING checks |
| **Step N** | At least one guided step with verification |
| **Verification** | Final all-green check |
| **Challenge** | Hands-on task |

## Hard Constraints -- MANDATORY

These apply to every module, every file, every commit in this repo. Violating them is a blocking review issue.

1. **Local Ollama backend ONLY.** Never reference, scaffold, document, or hint at OpenAI, public Gemini, Anthropic API passthrough, Ollama Cloud, or any other remote provider. The course's value proposition is local-only, AUP-safe inference.

2. **LiteLLM proxy binds to 127.0.0.1 only** -- never 0.0.0.0.

3. **LiteLLM version >= 1.83.** PyPI versions 1.82.7 and 1.82.8 were compromised. Always pin above them.

4. **No external-provider scaffolding.** Config files must not contain entries for non-Ollama providers, even commented out.

5. **Extensible by appending.** Structure configs so a future module can add another local model by appending one `model_list` entry. No external-provider scaffolding.

## Conventions

- **No conventional-commit prefixes** in commit messages or PR titles. Write plain English.
- **No emojis** in any written output.
- **Preflight checks** use the EXISTS/MISSING pattern: audit first, skip what's green.
- **Read-only checks** run automatically; system-modifying actions use the `!` prefix in module instructions.

## Marking a Module as NEW

Add `<!-- NEW -->` as the last line of the module file. Remove it when the module is no longer new. The catalog detects this marker automatically.
