# Module NN -- TITLE

Estimated time: X minutes
Prerequisites: LIST OR "None"

ONE SENTENCE: what this module does and what the user will have when done.

## Orientation

Print this once at the start:

```
You're setting up TOPIC.
This takes about X minutes.

We'll set up:
  1. First thing
  2. Second thing
  3. Third thing

You'll need: LIST WHAT THE USER NEEDS BEFORE STARTING.
```

## Progress Tracking

On module start, write a progress marker:

```bash
mkdir -p ~/.claude/courseware-progress && date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/NN.started
```

## Preflight

Audit current state before doing anything. Each check prints EXISTS or MISSING.

```bash
# Check 1 -- DESCRIPTION
COMMAND && echo "EXISTS: LABEL" || echo "MISSING: LABEL"

# Check 2 -- DESCRIPTION
COMMAND && echo "EXISTS: LABEL" || echo "MISSING: LABEL"
```

Print a summary of what was found. Skip any step below where the item already exists and is valid.

## Step 1 -- STEP TITLE

Skip if CONDITION.

EXPLANATION OF WHAT THIS STEP DOES AND WHY.

If missing, tell the user:
```
INSTRUCTIONS FOR THE USER.
For system-modifying commands, prefix with !

  ! COMMAND_USER_RUNS
```

Verify:
```bash
VERIFICATION_COMMAND
```

## Step 2 -- STEP TITLE

REPEAT PATTERN: skip condition, explanation, user instructions, verification.

## Verification

Run all preflight checks again as PASS/FAIL:

```bash
PASS=0
TOTAL=N

COMMAND && { echo "PASS: LABEL"; PASS=$((PASS+1)); } || echo "FAIL: LABEL"
# repeat for each check

echo ""
echo "$PASS/$TOTAL checks passed."
```

If all pass, print:
```
All checks passed. SUMMARY OF WHAT IS NOW CONFIGURED.
```

If any fail, tell the user which step to re-run.

## Challenge

```
DESCRIBE A HANDS-ON TASK.

Tell me:
  1. First thing to report
  2. Second thing to report
```

## Challenge Verification

DESCRIBE HOW TO VERIFY THE USER'S ANSWERS.
Use bash commands or ask the user to show output.

If successful, write the completion marker:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ > ~/.claude/courseware-progress/NN.done
```

Then print:
```
Module NN complete.

SUMMARY OF WHAT THE USER CAN NOW DO.

Next module: /learn-NN+1-NEXT-TOPIC

Questions or feedback? Open an issue in this repo.
```

---

## Local-LLM Reminders

When writing modules for this course, remember:

- **Local Ollama backend ONLY.** No references to remote providers.
- **LiteLLM >= 1.83.** Versions 1.82.7 and 1.82.8 were compromised.
- **Proxy binds to 127.0.0.1 only.**
- **Config extensible by appending.** One `model_list` entry per model.
