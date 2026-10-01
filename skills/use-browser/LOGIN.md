# Log in

Login state lives in one file per repo, inside the git dir, so git never tracks it and every worktree shares it. Print its absolute path and use that literal path below:

```bash
echo "$(git rev-parse --path-format=absolute --git-common-dir)/playwright-auth.json"
```

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
