# Local LLM Courseware

Before displaying the catalog, run the progress scan.

## Progress Scan

Run this silently to detect completion/in-progress state:

```bash
PROGRESS_DIR="$HOME/.claude/courseware-progress"
if [ -d "$PROGRESS_DIR" ]; then
  for f in modules/[0-9]*.md; do
    n=$(basename "$f" | grep -o '^[0-9]*')
    if [ -f "$PROGRESS_DIR/$n.done" ]; then
      echo "DONE:$n"
    elif [ -f "$PROGRESS_DIR/$n.started" ]; then
      echo "IN_PROGRESS:$n"
    fi
  done
fi
```

Use the output to add status tags to the catalog. See the tag rules below.

## Catalog

Print the catalog using markdown (NOT inside a code block).

**Tag rules** (append after the duration, in this priority order):
- If the progress scan printed `DONE:NN` for a module, show `done`
- If the progress scan printed `IN_PROGRESS:NN`, show `in progress`
- If the module file contains `<!-- NEW -->`, show **NEW**
- Otherwise, no tag

## Local LLM Courseware

### Setup & Foundation
`01`  LiteLLM + Ollama Setup . 20 min

## Footer

After the catalog, print:

> **[COUNT] modules available.** Modules marked **NEW** were recently added.
>
> Where [COUNT] is computed by counting `modules/[0-9]*.md` files excluding TEMPLATE.md:
> ```bash
> ls modules/[0-9]*.md | wc -l | tr -d ' '
> ```
>
> Pick a **number** to jump into a module. You can also type `/learn-` then Tab to see all modules, or `/preflight` to check your prerequisites.
>
> Questions? Open an issue in this repo.

## Module Routing

When the user picks a module, tell them to run the corresponding command:

| Module | Command |
|--------|---------|
| 01 | `/learn-01-litellm-ollama-setup` |
