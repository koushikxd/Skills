---
name: use-browser
description: Drive a headless browser with playwright-cli. Use when a task needs a real browser: verifying a change you made to a web UI, taking screenshots, or reading and acting on pages that need JavaScript or a login.
---

# Browser

You drive your own headless browser through `playwright-cli`. Each session is a fresh in-memory profile, so parallel agents never share state, and the user's own browser stays untouched.

`playwright-cli --help` and `playwright-cli <command> --help` are the command reference. This file is the workflow: start a session, work the page, close.

## Running commands

Run every `playwright-cli` call as its own bare command, with the executable as the first word and literal paths as arguments. The Claude Code sandbox blocks Chromium, so `playwright-cli *` is excluded from it by pattern. A call wrapped in `cd … &&`, a variable assignment, or a pipe misses the pattern and fails with `EPERM`. Resolve paths with plain shell commands first, then paste the values in.

## 1. Start a session

Write the browser choice to `.playwright-cli/browser.json` in the repo root, the CLI's own output dir, and keep that dir out of git:

```bash
mkdir -p .playwright-cli
EXCLUDE="$(git rev-parse --git-common-dir)/info/exclude"
grep -qx '.playwright-cli/' "$EXCLUDE" || echo '.playwright-cli/' >> "$EXCLUDE"
```

If `.playwright-cli/browser.json` already exists, skip to the open command at the end of this step.

Otherwise, take the first Chromium-family app installed:

```bash
for app in "Helium" "Google Chrome" "Brave Browser" "Microsoft Edge" "Chromium"; do
  exe="/Applications/$app.app/Contents/MacOS/$app"
  [ -x "$exe" ] && echo "$exe" && break
done
```

Write its path into the config:

```json
{ "browser": { "browserName": "chromium", "launchOptions": { "executablePath": "<path printed above>" } } }
```

Stock Firefox and Safari do not work: Playwright drives only its own patched builds of those engines. With no Chromium-family app installed, ask the user to run `playwright-cli install-browser chromium`, then write `{ "browser": { "browserName": "chromium" } }`.

Pick one session name for the whole task, unique to you (the branch name works), and open it:

```bash
playwright-cli -s=<name> open <url> --config=.playwright-cli/browser.json
```

For a local app, take the URL from the running dev server: the port in `package.json` scripts or the framework config, confirmed with `curl -sI <url>`. If nothing answers, ask the user to start it.

Done when: `open` prints a page status. If it prints an error, the config is wrong: fix it before going on.

If the page shows a sign-in form or redirects to a login URL, read [LOGIN.md](LOGIN.md) and log in before going on.

## 2. Work the page

`snapshot` gives the page as text with element refs; act on those refs (`click e15`, `fill e5 "text"`) and snapshot again to see the result. Read page content from snapshots; take a screenshot when the question is visual: layout, styling, how a change looks.

Verifying a change you made means observing the new behaviour in the browser: the element renders, the click does what the code says, the error state appears. Exercise the change itself, not just the page it lives on.

For screenshots, frame each on its subject: `resize` for viewport width, `set-color-scheme dark` for themes, `screenshot e5` for one element.

```bash
playwright-cli -s=<name> screenshot --filename=.playwright-cli/<descriptive-name>.png
```

Open every PNG with the Read tool and look at it. A spinner, an error overlay, or the login page is a failed capture: wait for the content, fix the state, retake.

To attach screenshots to a pull request, read [PR.md](PR.md).

Done when: you have observed in the browser everything the task asked about, and looked at every screenshot you took.

## 3. Close

```bash
playwright-cli -s=<name> close
```

Close your own session only: `close-all` and `kill-all` end other agents' browsers too.
