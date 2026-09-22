---
name: use-codex
description: Spawn one or more OpenAI Codex CLI subagents from bash to offload context-heavy work from the parent Claude. Use when the user says "use codex", "build with codex", "ask codex", "get a codex second opinion", "compare codex vs claude", "have two agents try it", or otherwise asks to delegate or parallelize work to Codex. Also use whenever a user asks for a large coding task: research, multi-file refactors, large codebase analysis.
---

# Codex Subagent Skill

Spawn autonomous Codex CLI subagents to offload context-heavy work. Subagents burn their own tokens and return only their final message, so the parent's context stays clean.

**Golden Rule:** If task + intermediate work would add 3,000+ tokens to parent context, use a subagent.

## Instructions

When invoking this skill:

1. **Clarify intent.** Figure out what the user wants. If unclear, ask. Inline args ("use codex to review my auth plan") usually carry the intent, so infer from them. Common buckets: second-opinion code review, refactor, plan validation, implementation of features, fresh perspective on a stuck bug, parallel comparison of approaches.
2. **Pick model + reasoning by task complexity** (see Model + Reasoning Selection). Default: `gpt-6-luna` at `high`.
3. **Preflight before fan-out.** Before launching multiple agents, confirm codex is authenticated and not rate-limited: run `command codex login status` (or one cheap low-effort probe) first. Hitting a 401 or rate limit five agents into a fan-out wastes every agent already launched.
4. **Spawn the subagent** using the canonical invocation (Basic Usage), with the sandbox disabled (Invocation Mechanics). Pipe long prompts via stdin. For anything expected to run more than a minute, or any fan-out, run it in the background (see Background Fan-Out) so the parent keeps working.
5. **Act autonomously while it runs.** Don't ask for permission mid-flight; the parent only sees the final result, so mid-task pauses waste tokens. Pause only for genuinely destructive operations (data loss, external impact, security).
6. **Monitor, don't fire-and-forget.** Check completion, retry on failure, answer follow-ups if blocked. Feel free to run multiple sequential or parallel codex subagents.
7. **Verify independently, mandatory.** Codex reporting success is a claim, not a result (see Verification). Never relay its summary as verified fact, and never dispatch dependent work on top of an unverified result.
8. **Present results, don't dump them.** Summarize what Codex said in your own words, surface concrete code changes or recommendations, and leave the next move to the user. Subagent output is *input* for your synthesis. Keep your response to the user concise and specific.

## Model + Reasoning Selection

Scale the model AND reasoning effort to task complexity.

The GPT-6 family supersedes GPT-5.6. There is no GPT-6 Terra; Luna replaces it as the workhorse. Ranked by raw capability:

1. **`gpt-6-astra`**: the strongest. Roughly 4-5x Sol's cost per task.
2. **`gpt-6-sol`**: the heavy hitter. Beats `gpt-5.6-sol` at every price point.
3. **`gpt-6-luna`**: the workhorse, cheaper and better than `gpt-5.6-terra`. **This is the default.**

Two calibration points (from OpenAI's AutomationBench cost/score curves) drive every routing decision:

- **Luna at its top effort ≈ Sol at its lowest, at about 1/5 the cost.** Exhaust Luna `xhigh` before reaching for Sol.
- **Sol at its best beats Astra at its cheapest, at about 1/4 the cost.** Astra is the ceiling. Reserve it for problems that have already defeated a Sol `xhigh` pass. Don't open with it.

GPT-6 also fabricates far less on OpenAI's coding-deception eval (Astra 0.5%, Sol 1.3%, Luna 2.8%, vs ~10% for GPT-5.6). Not zero: Verification stays mandatory.

Luna on `medium`/`high`/`xhigh` is the right call for the overwhelming majority of delegated work. Use judgement; the table is a starting point, not a rule.

| Complexity | Model | Reasoning |
|---|---|---|
| Trivial (lookup, one-liner, format fix) | `gpt-6-luna` | `low` |
| Simple/short (small edit, quick search, basic script) | `gpt-6-luna` | `medium` |
| Standard implementation (feature, refactor, multi-file) | `gpt-6-luna` | `high` |
| Hard/long-horizon (architecture, big migration, deep review) | `gpt-6-luna` | `xhigh` |
| Brutal (stuck bug that survived a Luna `xhigh` pass, subtle correctness, gnarly concurrency) | `gpt-6-sol` | `high`, then `xhigh` |
| Ceiling (survived a Sol `xhigh` pass) | `gpt-6-astra` | `high`, then `xhigh` |

Escalation ladder when a pass comes back wrong: Luna `high` → Luna `xhigh` → Sol `high` → Sol `xhigh` → Astra `high` → Astra `xhigh`. Escalate on *evidence of failure*, not on a hunch that the task looks hard.

GPT-6 adds `max` (all three) and `ultra` (Sol, Astra only) above `xhigh`. No calibration data exists for them yet, so don't route to them by default.

Set via `-m <model> -c 'model_reasoning_effort="<effort>"'`. `gpt-5.6-sol` and `gpt-5.5` remain valid fallbacks if a GPT-6 variant errors.

To list what the local CLI thinks is available:

```bash
python3 -c "import json,os; d=json.load(open(os.path.expanduser('~/.codex/models_cache.json'))); print(d['fetched_at']); [print(m['slug']) for m in d['models']]"
```

Note the `fetched_at` date. This cache goes stale and routinely omits models that work fine. A model missing from the cache is not evidence it's unavailable; a one-line low-effort probe is the only real test.

## Invocation Mechanics

Two things hold for every call below. Both failures masquerade as something else, so get them right the first time.

**Always invoke as `command codex`.** `codex` is often aliased in an interactive shell, for example to `clear && codex --yolo`. A bare `codex exec` then expands to `clear && codex --yolo exec ...`, which injects terminal escape sequences (`\x1b[H\x1b[J`) into stdout and corrupts any `--json` parsing. Every example below uses `command codex`.

**Claude Code parent: run `codex exec` outside the Bash sandbox.** `codex exec` starts its own app-server process and needs filesystem access the sandbox denies. Sandboxed, every run dies at startup with `Error: failed to initialize in-process app-server client: Operation not permitted (os error 1)`. This is a fixed property of the tool, not a transient failure, so a sandboxed attempt is pure waste. Set `dangerouslyDisableSandbox: true` on the *first* attempt of *every* `codex exec` Bash call, including background and fan-out calls. The harness default of retrying inside the sandbox first applies to commands that might work there; this one cannot.

Note that `command codex login status` runs fine sandboxed, so a green preflight says nothing about whether `exec` will start.

## Intelligent Prompting

Subagents only see what you give them. Be specific:

1. **Context**: what they're analyzing, where it lives.
2. **Objectives**: numbered, concrete.
3. **Constraints**: what to focus on, what to ignore.
4. **Output format**: exact shape the parent wants back.
5. **Success criteria**: when the task is done.

Template:

```
[TASK CONTEXT]
You are researching/analyzing/coding [TOPIC/EXPLANATION].

[OBJECTIVES]
1. ...
2. ...

[CONSTRAINTS]
- Focus on: ...
- Ignore: ...

[OUTPUT FORMAT]
Return: ...

[SUCCESS CRITERIA]
Complete when: ...
```

### Good vs. Bad Prompts

Vague prompts produce vague work; specific prompts produce useful work.

**Research**

❌ "Research authentication"

✅ "Research authentication in this Next.js codebase. Focus on: 1) Session management strategy (JWT vs session cookies), 2) Auth provider integration (NextAuth, Clerk, etc), 3) Protected route patterns. Check /app, /lib/auth, and middleware files. Return architecture summary with code examples."

**Web search / docs lookup**

❌ "Search for Codex SDK"

✅ "Find the most recent Codex SDK documentation using the web and summarize key updates. Focus on: 1) Installation/quickstart, 2) Core API methods and parameters, 3) Breaking changes or deprecations. Prioritize official OpenAI docs and release notes. Return a concise summary with citations."

**API discovery**

❌ "Find API endpoints"

✅ "Find all REST API endpoints in this Express.js app. Look in /routes, /api, and /controllers directories. For each endpoint document: method (GET/POST/etc), path, auth requirements, request/response schemas. Return as markdown table."

## Basic Usage

**Default: pipe the prompt via stdin using `-` as the positional argument.** Inline string prompts work for short ones, but anything with newlines, quotes, backticks, or `$` should be piped to avoid shell-escaping bugs.

Canonical invocation (capture output to file, Luna at high reasoning):

```bash
cat <<'EOF' | command codex exec --yolo --skip-git-repo-check \
  -m gpt-6-luna -c 'model_reasoning_effort="high"' \
  -o /tmp/codex-result.txt -
[TASK CONTEXT]
You are analyzing /path/to/repo.

[OBJECTIVES]
1. Do X
2. Do Y

[OUTPUT FORMAT]
Return: path - purpose
EOF

result=$(cat /tmp/codex-result.txt)
```

**Variants** (only what changes from the canonical form):

- **Trivial task:** swap to `-c 'model_reasoning_effort="low"'`, model stays Luna.
- **Hard/long-horizon task:** raise to `-c 'model_reasoning_effort="xhigh"'`, model stays Luna.
- **Brutal task, after a Luna `xhigh` pass already failed:** swap to `-m gpt-6-sol -c 'model_reasoning_effort="high"'`.
- **Ceiling task, after a Sol `xhigh` pass already failed:** swap to `-m gpt-6-astra -c 'model_reasoning_effort="high"'`.
- **Machine-parsable output:** swap `-o /tmp/codex-result.txt` for `--json`, then pipe through `jq -r 'select(.event=="turn.completed") | .content'`. Prefer `-o` whenever possible: it skips JSON parsing and avoids terminal truncation on long outputs.

## Parallel Subagents

Spawn multiple subagents for independent tasks: research two topics in parallel, or compare two approaches to the same problem. Each writes to its own `-o` file; `wait` for all, then read:

```bash
cat <<'EOF' | command codex exec --yolo --skip-git-repo-check \
  -m gpt-6-luna -c 'model_reasoning_effort="high"' -o /tmp/agent-a.txt - &
Approach A: solve [problem] using [strategy A]. Return diff + rationale.
EOF

cat <<'EOF' | command codex exec --yolo --skip-git-repo-check \
  -m gpt-6-luna -c 'model_reasoning_effort="high"' -o /tmp/agent-b.txt - &
Approach B: solve [problem] using [strategy B]. Return diff + rationale.
EOF

wait
RESULT_A=$(cat /tmp/agent-a.txt)
RESULT_B=$(cat /tmp/agent-b.txt)
```

The canonical compare-two-approaches pattern: give both subagents the same problem under different framings, wait, then synthesize the better answer in the parent.

## Background Fan-Out (Claude Code parent)

When the parent is Claude Code, prefer its background-task machinery over shell `&`/`wait` for long runs and fan-outs: issue one Bash call per agent with `run_in_background: true`, each writing to its own `-o` file. The parent keeps working; the harness sends a task notification when each agent exits, and interim progress can be checked by Reading the task's output file.

- Give each agent a distinct, descriptive `-o` path (`/tmp/codex-server.txt`, `/tmp/codex-ios.txt`). The filename is your only label when notifications arrive out of order.
- On each notification: read the `-o` file, verify the result (see Verification), then decide whether to dispatch the next slice, a fix agent, or nothing.
- Fan out only across genuinely independent work (different repos, packages, or modules). Don't launch agents whose inputs depend on another agent's unfinished output.
- Blocking `&`/`wait` is fine for short parallel pairs where the parent has nothing else to do.

## Git Hygiene for Multiple Agents

Codex agents run `--yolo`: no sandbox, no approval prompts, full filesystem and network access. They will happily commit over each other. Rules:

- **One repo = one writer at a time.** Parallel writers are safe only across different repos/packages, or give each agent its own git worktree.
- **Tell each agent the tree state it will find**: which branch to use, any uncommitted changes it must preserve, and explicitly whether it may commit/push or must leave changes uncommitted for the parent to review.
- **If another agent or session may have committed meanwhile**, instruct the agent to fetch and rebase before committing and to keep its change minimal.
- After any agent that commits, the parent checks `git log`/`git status` itself to confirm what actually landed and that nothing else was clobbered.

## Verification (mandatory)

Codex's final message saying "done" is a claim, not a result. After **every** run, the parent independently verifies before reporting to the user or building on the result:

- **Code changes:** inspect `git diff`/`git status`, confirm the change is in the files it names.
- **Builds/sites:** rerun the build or check the built artifact for the specific expected change.
- **Deploys/live content:** fetch the live URL with a cache-buster (`?v=$(date +%s)`) and confirm the specific change is visible.
- **Claims about code ("X is handled in Y"):** spot-check the cited file/line.

If verification fails, dispatch a fix agent with the evidence. Don't re-report the original claim.

## Sequential Subagents

When the parent is driving a multi-step engagement, each next call is a *judgment* based on what just came back, not a mechanical retry. Spawn one subagent, read its output, decide what to do next (next slice of the work, a review pass, a redo, a different angle, a verification step), spawn the next one, repeat. The parent stays in the driver's seat the whole time:

```bash
# Step 1: dispatch the first subagent
cat <<'EOF' | command codex exec --yolo --skip-git-repo-check \
  -m gpt-6-luna -c 'model_reasoning_effort="high"' \
  -o /tmp/codex-step-1.txt -
Implement the auth middleware in src/middleware/auth.ts per the spec at docs/auth.md.
Return: summary of changes + files touched.
EOF

# Step 2: parent reads /tmp/codex-step-1.txt, decides what's needed next.
# Could be: next part of the task, a critique pass, a redo, a verification, whatever fits.
# Always use a quoted heredoc (<<'EOF') for the template and `cat` the prior
# output on stdin. Never interpolate $STEP_N into an unquoted heredoc:
# Codex output frequently contains backticks and $() sequences (code blocks,
# command examples), and bash would execute those during expansion.
{
  cat <<'EOF'
Review the auth middleware changes summarized below for security holes
(token leakage, timing attacks, missing CSRF). Return: issues + line refs.

[PRIOR WORK]
EOF
  cat /tmp/codex-step-1.txt
} | command codex exec --yolo --skip-git-repo-check \
  -m gpt-6-luna -c 'model_reasoning_effort="high"' \
  -o /tmp/codex-step-2.txt -

# Step 3: parent reads /tmp/codex-step-2.txt, dispatches a fix pass, or moves
# on to the next slice of work entirely. Keep going until the parent decides done.
```

The shape: **call → read → decide → call → read → decide**. Each prompt is composed fresh in the parent based on what just landed. There's no fixed loop count and no shared template: every step can be a totally different task (implement, review, refactor, sanity-check, redo from scratch). Feed prior outputs into later prompts whenever the next subagent needs that context to do its job.

## Resuming a session (follow-ups)

**Never spawn a fresh session to iterate on work this CLI already did.** Resume the existing one so it keeps its context and you don't re-pay the harness-token cost:

```bash
command codex exec resume --last -m <model> -c 'model_reasoning_effort="<effort>"' "<follow-up>"
# or by id: command codex exec resume <SESSION_ID> ...
```

The session id is printed at the start of every run (`session id: 019f9ce2-...`) and is also in the `--json` stream. Capture it from the first run and prefer resuming by explicit id: `--last` is racy when several sessions exist, and a fan-out always creates several.

## Quota failover

If Codex is rate-limited or out of quota, don't silently downgrade the task. Options in order:

1. **Drop a tier within the family.** Sol `xhigh` → Sol `high` → Luna `high` → Luna `low`. Cheaper reasoning often clears a soft rate limit.
2. **Fall back to the previous generation**: `gpt-5.6-sol`, then `gpt-5.5`.
3. **Tell the user and stop.** A blown quota mid-fan-out means some agents completed and some didn't. Report exactly which `-o` files have real content before deciding anything else.
