# skills

A collection of [Agent Skills](https://skills.sh) for Claude Code and other agents.

## Install

Install every skill in this repo:

```bash
npx skills add koushikxd/skills
```

Or install a single skill by name:

```bash
npx skills add koushikxd/skills --skill deep-research
```

## Skills

| Skill | Description |
|-------|-------------|
| [`deep-research`](skills/deep-research/SKILL.md) | Right-sized, web-backed research that ends in a design or implementation decision. Use when weighing options before building. |
| [`create-design-md`](skills/create-design-md/SKILL.md) | Extract a project's real design tokens into a `DESIGN.md` spec so other agents can reproduce its UI exactly. |
| [`debrief`](skills/debrief/SKILL.md) | Understand what just happened. Opens an HTML page explaining the session's work in plain English, with root causes, flow diagrams, and before/after. Invoke by hand with `/debrief`. |
| [`babysit-pr`](skills/babysit-pr/SKILL.md) | Watch a pull request until its automated reviewers go quiet. Polls for new bot comments, verifies each claim against the code, fixes what deserves fixing, pushes, repeats. |
| [`use-codex`](skills/use-codex/SKILL.md) | Delegate context-heavy work to OpenAI Codex CLI subagents from bash. Covers model and reasoning selection, parallel fan-out, session resume, and mandatory verification. |
| [`view-tweet`](skills/view-tweet/SKILL.md) | Read X/Twitter post links through the Grok CLI and bring back the verbatim text, full thread, quoted post, media, and replies as context. |

## Adding more skills

Drop a new folder under `skills/` containing a `SKILL.md` with `name` and `description` frontmatter:

```
skills/
└── <skill-name>/
    └── SKILL.md
```

The `npx skills` CLI discovers it automatically — no registry submission needed.
