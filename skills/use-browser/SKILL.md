---
name: use-browser
description: "Drive a headless browser with agent-browser. Use when a task needs a real browser: verifying a change you made to a web UI, taking screenshots, or reading and acting on pages that need JavaScript or a login."
---

# Browser

You drive your own headless browser through `agent-browser`. Each named session is its own browser with a fresh profile, so parallel agents never share state, and the user's own browser stays untouched.

`agent-browser --help` and `agent-browser <command> --help` are the command reference. This file is the workflow: start a session, work the page, close.

## Running commands

Run every `agent-browser` call as its own bare command: the executable as the first word, `--session <name>` on every call, literal paths as arguments. The Claude Code sandbox blocks Chromium and agent-browser's socket dir, so `agent-browser *` is excluded from it by pattern. A call wrapped in `cd … &&`, a variable assignment, or a pipe misses the pattern and fails with `Operation not permitted`. Exported env vars are gone by the next call, which is why the session rides on every call as a flag. Resolve paths with plain shell commands first, then paste the values in.

## 1. Start a session

Keep screenshots in `.agent-browser/` in the repo root, out of git:

```bash
mkdir -p .agent-browser
EXCLUDE="$(git rev-parse --git-common-dir)/info/exclude"
grep -qx '.agent-browser/' "$EXCLUDE" || echo '.agent-browser/' >> "$EXCLUDE"
```

The browser choice lives once per machine, in `~/.agent-browser/config.json`. If that file already sets `executablePath`, skip to the open command at the end of this step.

Otherwise, take the first Chromium-family app installed:

```bash
for app in "Helium" "Google Chrome" "Brave Browser" "Microsoft Edge" "Chromium"; do
  exe="/Applications/$app.app/Contents/MacOS/$app"
  [ -x "$exe" ] && echo "$exe" && break
done
```

Write its path into the config with your file-writing tool (the sandbox blocks shell writes to `~/.agent-browser`), keeping any keys already there:

```json
{ "executablePath": "<path printed above>" }
```

agent-browser drives Chromium over CDP, so stock Firefox and Safari do not work. With no Chromium-family app installed, ask the user to run `agent-browser install` (downloads Chrome for Testing) and skip the config.

Pick one session name for the whole task, unique to you (the branch name works), and open it:

```bash
agent-browser --session <name> open <url>
```

For a local app, take the URL from the running dev server: the port in `package.json` scripts or the framework config, confirmed with `curl -sI <url>`. If nothing answers, ask the user to start it.

Done when: `open` prints `✓` with the page title. If it prints an error, the browser config is wrong: fix it before going on.

If the page shows a sign-in form or redirects to a login URL, read [LOGIN.md](LOGIN.md) and log in before going on.

## 2. Work the page

`snapshot -i` lists the interactive elements with refs; act on those refs (`click @e15`, `fill @e5 "text"`). Refs go stale when the page changes, so snapshot again before the next action.

`-i` keeps only what you can act on. The plain text a check reads, such as status messages, counts, error copy, or the result of the click you just made, appears only in a full `snapshot`. Scope it to the region you are checking with `-s "<css selector>"`.

Actions return the moment the event fires, before the page reacts. After a click that navigates or fetches, `wait` on the outcome you expect (`wait --text "Saved"`, `wait --url "**/settings"`), then read the page. A wait that times out means the outcome never happened: treat it as a failed check.

Read page content from snapshots; take a screenshot when the question is visual: layout, styling, how a change looks.

Verifying a change you made means observing the new behaviour in the browser: the element renders, the click does what the code says, the error state appears. Exercise the change itself, not just the page it lives on.

For screenshots, frame each on its subject: `set viewport <w> <h>` for width, `set media dark` for themes, `screenshot @e5 <path>` for one element.

```bash
agent-browser --session <name> screenshot .agent-browser/<descriptive-name>.png
```

Open every PNG with the Read tool and look at it. A spinner, an error overlay, or the login page is a failed capture: wait for the content, fix the state, retake.

To attach screenshots to a pull request, read [PR.md](PR.md).

Done when: you have observed in the browser everything the task asked about, and looked at every screenshot you took.

## 3. Close

```bash
agent-browser --session <name> close
```

Close your own session only: `close --all` ends other agents' browsers too.
