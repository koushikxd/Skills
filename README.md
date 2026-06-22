# skills

A collection of [Agent Skills](https://skills.sh) for Claude Code and other agents.

## Install

Install every skill in this repo:

```bash
npx skills add koushikxd/skills
```

Or install a single skill by name:

```bash
npx skills add koushikxd/skills --skill research
```

## Skills

| Skill | Description |
|-------|-------------|
| [`research`](skills/research/SKILL.md) | Right-sized, web-backed research that ends in a design or implementation decision. Use when weighing options before building. |

## Adding more skills

Drop a new folder under `skills/` containing a `SKILL.md` with `name` and `description` frontmatter:

```
skills/
└── <skill-name>/
    └── SKILL.md
```

The `npx skills` CLI discovers it automatically — no registry submission needed.
