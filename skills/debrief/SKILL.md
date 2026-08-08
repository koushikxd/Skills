---
name: debrief
description: Understand what just happened. Opens an HTML page explaining the session's work in plain English, with root causes, flow diagrams, and before/after.
disable-model-invocation: true
argument-hint: "(optional) what to focus on"
---

# Debrief

A **debrief** is an account of finished work, given to someone who was not there. That is who you
write for: the user watched you work but does not know what you did, and wants to understand it
without reading the diff.

Cover the whole session unless the user's argument narrows it.

Every sentence in the page needs a **receipt**: a diff hunk, a file you read this run, or a URL.
Where you have no receipt, write the doubt into the page in plain words. Simple language makes a
guess read like a fact, which is the one way this page can do harm.

## Step 1 — Ground truth

Two commands, then read what they point at:

```bash
git status --short && git diff HEAD --stat
git log --oneline -30
```

Uncommitted work is `git diff HEAD`. Committed work is the commits you made this session; you wrote
those messages, so you can recognise them. The union is the change set.

No repo, or an empty change set, means the session was research or planning. That is a valid
debrief: skip Steps 2 and 3, build one `research` work item, and say at the top that no code
changed.

**Done when** every path in `git status --short` appears in your change set.

## Step 2 — Split into work items

A session is not one thing. Split the change set into **work items** by intent, not by file, and
give each a type. The type decides which blocks it carries:

| Type          | Carries                            | Never carries               |
| ------------- | ---------------------------------- | --------------------------- |
| `bug`         | `root-cause`, `diff`               | —                           |
| `feature`     | `walkthrough`, `diagram`           | `diff` (there is no before) |
| `improvement` | `diff` before and after, `callout` | —                           |
| `research`    | `findings` with inline citations   | `diff`                      |

Past six items, group the small ones into one item called "Smaller changes", one line each.

**Done when** every changed file belongs to exactly one work item, and every item has a type.

## Step 3 — Read the code as it now stands

The diff shows what moved. It does not show how the thing works, which is what the user asked for.
Read the final state of the ten most-changed files, by hunk count. Work from the diff alone for the
rest, and list that in `not-covered`.

**Done when** every code work item has a named entry point: the file and the function where its
behaviour starts.

## Step 4 — Write the body

Read `REFERENCE.md` for the block markup. Write the body to `$TMPDIR/debrief-body.html`, in this
order: `hero`, `tldr`, `file-map`, work items, `timeline` if the session wandered, `glossary` if you
used codebase words, `check-yourself`, `not-covered`.

### Language

Write ASD-STE100 Simplified Technical English.

Gloss every codebase and domain word in the same sentence you first use it: "the tenant, meaning one
customer company". Explain mechanisms, not vocabulary: "it waits until the file finishes
downloading", not "it awaits the promise".

Every sentence must be true of this session only. If a sentence would survive being pasted into a
different debrief, delete it.

### Budget

`tldr` is one paragraph. A work item is three to six sentences plus its blocks. Put the honest
reading time in `hero`.

**Done when** the body holds no sentence you cannot point to a receipt for.

## Step 5 — Ship it

The body is the only file you write. The command below copies the template byte for byte and splices
the body into it.

```bash
OUT="$TMPDIR/debrief-$(basename "$PWD")-$(date +%Y%m%d-%H%M).html"
python3 - "<skill dir>/assets/template.html" "$TMPDIR/debrief-body.html" "$OUT" "<title>" <<'PY'
import sys, pathlib
tpl, body, out, title = sys.argv[1:5]
html = pathlib.Path(tpl).read_text()
html = html.replace("<!-- CONTENT -->", pathlib.Path(body).read_text())
html = html.replace("<title>Debrief</title>", "<title>%s</title>" % title)
pathlib.Path(out).write_text(html)
PY
open "$OUT" && printf '%s' "$OUT" | pbcopy
```

Write to the repo root as `debrief-<date>.html` only if the user asks, and add it to `.gitignore`
first.

**Done when** the file is open in the browser and its path is on the clipboard.
