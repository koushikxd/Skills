---
name: use-browser
description: Drive a headless browser with playwright-cli to check a web UI, log into a local dev app, take screenshots, and attach them to a PR. Use after finishing UI work that should be seen, when the user asks to open, click through, or screenshot a page, or when a PR needs visual evidence.
---

# Browser

You drive your own headless browser through `playwright-cli`. Each session is a fresh in-memory profile, so parallel agents never share state. The user's own browser stays untouched: every session starts with `open`.

`playwright-cli --help` and `playwright-cli <command> --help` are the command reference. This file is the workflow: browser, login, capture, PR, close.

## Running commands

Run every `playwright-cli` call as its own bare command, with the executable as the first word and literal paths as arguments. The Claude Code sandbox blocks Chromium, so `playwright-cli *` is excluded from it by pattern. A call wrapped in `cd … &&`, a variable assignment, or a pipe misses the pattern and fails with `EPERM`. Resolve paths with plain shell commands first, then paste the values in.

## 1. Pick a browser

Write the choice to `.playwright-cli/browser.json` in the repo root, the CLI's own output dir, and keep that dir out of git:

```bash
mkdir -p .playwright-cli
EXCLUDE="$(git rev-parse --git-common-dir)/info/exclude"
grep -qx '.playwright-cli/' "$EXCLUDE" || echo '.playwright-cli/' >> "$EXCLUDE"
```

If `.playwright-cli/browser.json` already exists, skip to the verify command at the end of this step.

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

Pick one session name for the whole task, unique to you (the branch name works), and verify:

```bash
playwright-cli -s=<name> open about:blank --config=.playwright-cli/browser.json
```

Done when: `open` prints a page status. If it prints an error, the config is wrong: fix it before going on.

## 2. Log in

Find the app's URL from the running dev server: the port in `package.json` scripts or the framework config, confirmed with `curl -sI <url>`. If nothing answers, ask the user to start it.

Login state lives in one file per repo, inside the git dir, so git never tracks it and every worktree shares it. Print its absolute path and use that literal path below:

```bash
echo "$(git rev-parse --path-format=absolute --git-common-dir)/playwright-auth.json"
```

Skip this step for pages that need no login.

**State file exists**: load it into your open session, then go to an authed page.

```bash
playwright-cli -s=<name> state-load <auth path>
playwright-cli -s=<name> goto <authed url>
```

The first `goto` after `state-load` sometimes fails with `net::ERR_ABORTED`; run the same `goto` again.

Logged in means the snapshot shows the authed page, not a sign-in form or a redirect to a login URL. If it shows the login page, the session expired or the dev DB was reset: renew.

**No state file, or it expired**: renew, cheapest first.

1. **Repo recipe.** Look for a documented dev login: test credentials or a seed user in `AGENTS.md`, `CLAUDE.md`, the README, `.env.example`, or a seed script; a dev-only login or magic-link script. If one exists, log in with it in your session.
2. **Ask the user.** Open a visible window and hand it over:
   ```bash
   playwright-cli -s=login open <login url> --headed --config=.playwright-cli/browser.json
   ```
   Ask the user to log in with a dev or test account and reply when done. Save from that session, close it, and load the file into yours.

```bash
playwright-cli -s=<session that logged in> state-save <auth path>
```

Done when: `goto` on an authed page shows it logged in.

## 3. Capture

Navigate and interact with `snapshot` refs (`click e15`, `fill e5 "text"`). Frame each screenshot on the change: `resize` for viewport width, `set-color-scheme dark` for themes, `screenshot e5` for one element.

```bash
playwright-cli -s=<name> screenshot --filename=.playwright-cli/<descriptive-name>.png
```

Open every PNG with the Read tool and look at it. A spinner, an error overlay, or the login page is a failed capture: wait for the content, fix the state, retake.

Done when: every screenshot shows the change it claims to show, and you have looked at each one.

## 4. Attach to the PR

`gh pr create`, `gh pr comment`, and `gh pr edit` upload images with `--attach` (gh 2.99+). Text after `#` is the alt text. Reference the same path in the body to place an image inline; `gh` rewrites it to the uploaded URL.

```bash
gh pr create --title "..." --body-file body.md \
  --attach './.playwright-cli/settings-after.png#Settings page after the fix'
```

Images are capped at 10 MB. If `gh` rejects `--attach`, open the PR without images and tell the user to upgrade `gh`; the screenshots stay in `.playwright-cli/`.

## 5. Close

```bash
playwright-cli -s=<name> close
```

Close your own session only: `close-all` and `kill-all` end other agents' browsers too.
