---
name: use-codex
description: Spawn OpenAI Codex CLI agents from bash.
disable-model-invocation: true
---

# Use Codex

Each `codex exec` run is an autonomous agent that spends its own tokens and returns only its final message, so your context stays clean.

## Model and effort

Use the model and effort the user names. When they name none, omit `-m` (Codex uses its configured default) and set effort to `high`.

## Invocation

```bash
cat <<'EOF' | command codex exec --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check \
  [-m <model>] -c 'model_reasoning_effort="high"' \
  -o /tmp/codex-<name>.txt -
Context: what to work on and where it lives.
Objectives: 1. ... 2. ...
Output: the exact shape you want back.
Done when: ...
EOF
```

The agent sees only this brief, so it carries everything the task needs. Read the result from the `-o` file.

Gotchas:

- **`command codex`**, always. A shell alias on `codex` can inject terminal escapes into the output.
- **Claude Code: run unsandboxed.** Set `dangerouslyDisableSandbox: true` on the first attempt of every `codex exec` call; inside the sandbox it always dies at startup with `Operation not permitted`.
- **Literal `/tmp/` paths** for `-o` and any brief file. `$TMPDIR` resolves differently inside and outside the sandbox.
- **Feed prior output on stdin** (`{ cat <<'EOF' ... EOF; cat /tmp/codex-a.txt; } | command codex exec ...`). Expanding it into an unquoted heredoc executes any backticks or `$()` it contains.

## Parallel agents

Launch one background command per agent (`run_in_background: true` in Claude Code), each with its own descriptive `-o` file. Give agents that write code separate repos or git worktrees: one writer per tree.

## Follow-ups

Resume the session to keep its context: `command codex exec resume <SESSION_ID> "<follow-up>"`. The id prints at the start of each run; prefer it over `--last`, which is racy when several sessions exist.

## Verify

A Codex "done" is a claim. Before reporting or building on it, check the evidence yourself: `git diff` for code changes, rerun the build or test, spot-check cited file and line references. Then summarise the result for the user in your own words.
