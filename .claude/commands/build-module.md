# Build a New Courseware Module

You are building a new learning module for the local-llm-courseware project.

## Hard Constraints

Before writing any content, remember these constraints apply to every module:

1. **Local Ollama backend ONLY.** No references to remote providers.
2. **LiteLLM proxy binds to 127.0.0.1 only.**
3. **LiteLLM version >= 1.83** (1.82.7 and 1.82.8 were compromised).
4. **No external-provider scaffolding** in configs, even commented out.
5. **Extensible by appending** -- one `model_list` entry per model.

## Instructions

1. Ask: "Which module number and topic?" (e.g., "02 -- Model Comparison")
   If the user already specified it, skip this question.

2. Read `modules/TEMPLATE.md` for the exact structure to follow.

3. Ask: "Any specific source material I should read?" (e.g., a repo path, docs, or existing config)
   If the user provides a path, read it for content. If not, use your built-in knowledge of the tool.

4. Generate the module file at `modules/NN-TOPIC.md`.
   Follow TEMPLATE.md exactly. Fill in every placeholder.
   Enforce all hard constraints -- no remote provider references.

5. **Auto-generate the dispatcher** at `.claude/commands/learn-NN-TOPIC.md`:

   ```
   # TITLE

   DESCRIPTION.
   Estimated time: X minutes. Prerequisites: LIST.

   Read modules/NN-TOPIC.md but present it in phases:

   Phase 1: Read only the Orientation section. Present it.
            Ask: "Ready to check prerequisites?"

   Phase 2: Read only the Preflight section. Run the checks.
            Skip any step that passes. Report results.
            Ask: "Ready to start the walkthrough?"

   Phase 3: Read and present one Step at a time.
            After each step's verification passes, proceed to the next.
            Do not read ahead -- load each step only when needed.

   Phase 4: Read only the Verification section. Run all checks.
            Report results.

   Phase 5: Read only the Challenge and Challenge Verification sections.
            Present the challenge. After the user completes it, verify.

   Track progress in ~/.claude/courseware-progress/.
   ```

6. **Add to catalog** -- Ask the user: "Which section should this go in?"
   Show the section list from `courseware.md`. Insert the module entry
   in the correct section.

7. **Add to README** -- Insert a row in the correct section table in `README.md`.

8. Commit all generated files:

   ```bash
   git add modules/NN-TOPIC.md .claude/commands/learn-NN-TOPIC.md .claude/commands/courseware.md README.md
   git commit -m "add Module NN -- TITLE"
   ```

9. Report what was created and suggest `git push origin main` for the user to approve.
