---
name: research
description: Right-sized, web-backed research that ends in a design or implementation decision. Use when you're weighing options before building and ask "what's the best way to..." / "should I use X or Y" / "research X before I build it".
---

# Right-Sized Research

You research a decision the user has to make, then commit to a recommendation. Web-first,
not from memory — trained knowledge is stale for tools, versions, and benchmarks. Scale the
effort to the question, then commit — don't spawn a fleet to settle a one-liner.

## Rule 0: right-size before you spawn

Every sub-agent costs the user real budget. Spawn the **minimum** that genuinely needs
parallel coverage. Match the effort to the question:

| Question | What to do |
|---|---|
| Single fact / "latest version of X" / settled best practice | Answer directly. One web search if needed. **Zero sub-agents.** |
| One decision, clear options ("X or Y?") | 1-2 sub-agents, or just do the passes yourself. |
| Broad / architectural / new project, many unknowns | 3-5 sub-agents across distinct facets. |

When unsure, start smaller. You can always spawn another agent; you can't refund one.
Say in one line how big you sized it and why.

## Workflow

### 1. Scope (interview, sized)
Know what you're researching before you spend budget. Ask the user questions **one at a
time**, each with **your recommended answer**, until you can frame the research — then stop.
Volume scales with the question (Rule 0): a single-fact ask needs zero questions; a
new-project ask may need a few. Resolve a constraint by exploring the codebase instead of
asking whenever you can.

If you can't ask *good* questions without knowing the current landscape (unfamiliar or
fast-moving space), do **one quick orienting web search first**, then ask — don't fan out
yet. Stop interviewing the moment you have what would change the recommendation; never
interrogate past that.

Capture: the decision, the options, the context (demo vs production, scale, stack), and
what "good" looks like. A demo-first ask and a scale-later ask have different answers.

### 2. Research (sized per Rule 0)
Break the question into independent facets — only as many as it needs. Useful angles:
*how leaders do it · frameworks & maturity · tradeoffs for THIS context · recent evidence
(last ~12-18 months)*. For each facet that warrants a sub-agent, launch them **in one message**
(parallel). Give each a NARROW brief — its facet plus only the context it needs, not this
whole prompt. Every sub-agent must **search the live web** and return findings **with real
source URLs** — concrete tools, versions, numbers; an unsourced claim is a guess. Favor what
real production systems run over theory, and separate what's *popular* from what's *right for
this context*.

Done when every facet that could change the call has a sourced answer; flag anything still
speculative or dated rather than papering over it.

### 3. Adversarial review
Before you recommend, challenge your own front-runner. Take the strongest *rejected* option
and argue its case: **why this and not that?** Attack the leader on cost, lock-in, failure
modes, hidden complexity, and whether the evidence actually supports it. For broad research,
spawn one skeptic sub-agent for this; for small research, do it inline. If the challenge
lands, change the call. State what the strongest objection was and how it survived (or didn't).

### 4. Recommend
Lead with the call, then justify. Cover:
- **Recommendation** — which approach fits the stated constraints, and why; say whether you
  optimized for demo-first or scale-later
- **How to start** — the concrete first steps for the chosen path
- **Tradeoffs & risks** — what you give up; what bites later (informed by step 3)
- **When to revisit** — the signal that would change this decision (e.g. scale threshold)
- **Sources** — the strongest references behind the call

Weight conflicting findings by **cross-source agreement**: a claim several sources hit
independently beats a single blog post, and where they disagree, surface it rather than
paper over it. Say so.
