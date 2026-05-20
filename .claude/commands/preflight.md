# Preflight Check

Run a prerequisite check for Local LLM Courseware. This verifies the tools and access needed before starting modules.

## Run All Checks

Run these checks and print the results:

```bash
echo "Local LLM Courseware -- Preflight Check"
echo "========================================"
echo ""

PASS=0
TOTAL=8

# 1. Operating system
OS=$(uname -s)
if [ "$OS" = "Darwin" ]; then
  echo "PASS: macOS ($(sw_vers -productVersion 2>/dev/null || uname -r))"
  PASS=$((PASS+1))
else
  echo "FAIL: macOS required (found $OS) -- Linux support planned"
fi

# 2. Shell
if [ "$(basename "$SHELL")" = "zsh" ]; then
  echo "PASS: zsh default shell"
  PASS=$((PASS+1))
else
  echo "FAIL: zsh required (found $SHELL)"
fi

# 3. Homebrew
if command -v brew &>/dev/null; then
  echo "PASS: Homebrew ($(brew --version 2>/dev/null | head -1))"
  PASS=$((PASS+1))
else
  echo "FAIL: Homebrew not found -- install from https://brew.sh"
fi

# 4. Disk space
AVAIL_GB=$(df -g "$HOME" 2>/dev/null | tail -1 | awk '{print $4}')
if [ "${AVAIL_GB:-0}" -ge 20 ]; then
  echo "PASS: ${AVAIL_GB} GB free disk space"
  PASS=$((PASS+1))
else
  echo "FAIL: Need ~20 GB free (found ${AVAIL_GB:-unknown} GB)"
fi

# 5. Ollama
if command -v ollama &>/dev/null; then
  echo "PASS: Ollama installed"
  PASS=$((PASS+1))
else
  echo "INFO: Ollama not installed -- Module 01 will install it"
  PASS=$((PASS+1))
fi

# 6. Claude Code
if command -v claude &>/dev/null; then
  echo "PASS: Claude Code ($(claude --version 2>/dev/null || echo 'installed'))"
  PASS=$((PASS+1))
else
  echo "FAIL: Claude Code not found"
fi

# 7. No conflicting ANTHROPIC_API_KEY
if grep -q "^export ANTHROPIC_API_KEY" "$HOME/.zshrc" 2>/dev/null; then
  echo "WARN: ANTHROPIC_API_KEY exported globally in ~/.zshrc"
  echo "  This can interfere with profile switching."
  echo "  Consider removing the export and using profile-specific auth."
else
  echo "PASS: No global ANTHROPIC_API_KEY export"
  PASS=$((PASS+1))
fi

# 8. Existing Claude profiles (informational)
PROFILES=0
grep -q "claude-work" "$HOME/.zshrc" 2>/dev/null && PROFILES=$((PROFILES+1))
grep -q "claude-personal" "$HOME/.zshrc" 2>/dev/null && PROFILES=$((PROFILES+1))
if [ "$PROFILES" -ge 1 ]; then
  echo "PASS: $PROFILES existing Claude profile(s) found"
  PASS=$((PASS+1))
else
  echo "INFO: No existing Claude profiles found (recommended but not required)"
  PASS=$((PASS+1))
fi

echo ""
echo "$PASS/$TOTAL checks passed."
```

## After the Checks

Print a summary based on results:

If all pass:
> You're ready to go. Run `/courseware` to see the module catalog and start Module 01.

If Homebrew is missing:
> Homebrew is required. Install it first: https://brew.sh

If disk space is low:
> You need ~20 GB free for model weights. Free up space before starting Module 01.

If Claude Code is missing:
> Claude Code is required. See the Vertex AI courseware Module 01 for installation, or run: `npm install -g @anthropic-ai/claude-code`
