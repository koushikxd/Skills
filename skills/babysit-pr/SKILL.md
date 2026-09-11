---
name: babysit-pr
description: Babysit a pull request until its automated reviewers go quiet: poll for new bot comments, judge each one, fix what deserves fixing, push, repeat. Use when the user wants a PR watched, wants CodeRabbit/Greptile/Copilot/Codex review comments handled, or wants to keep iterating until the reviews are clean.
---

You are babysitting a PR. Stay with it, on your own, until the automated reviewers have nothing left to say.

## Setup

Resolve the PR and stay on that one:

```bash
gh pr view --json number,url,title,headRefName,body
```

Pass the number or URL if the user gave one. Check out the head branch locally. Read the PR body: its stated purpose is the boundary for every change you make from here.

## The loop

**1. Collect.** Bots post in three different places, so read all three:

```bash
gh pr view <n> --json comments --jq '.comments[] | {author: .author.login, createdAt, body}'
gh pr view <n> --json reviews  --jq '.reviews[]  | {author: .author.login, state, submittedAt, body}'
gh api repos/{owner}/{repo}/pulls/<n>/comments --jq '.[] | {id, user: .user.login, path, line, created_at, body}'
```

Keep only bot authors (`coderabbitai`, `greptile`, `cursor`, `copilot-pull-request-reviewer`, `gemini-code-assist`, `sourcery-ai`, `ellipsis-dev`, anything ending in `[bot]`) and only comments newer than your last pass. Record the newest timestamp you have seen; that is your cursor for the next round.

**2. Verify.** A bot comment is a claim, not a finding. Automated reviewers are confidently wrong often enough that agreeing by default is a mistake, and their prose gives you no way to tell a real bug from a hallucinated one. So check every claim against the code before you touch anything: open the file and line it cites, read the surrounding path, and confirm the failure it describes can actually happen. Where the claim is about behaviour, the cheapest proof is a test that fails now and passes after the fix.

Nothing gets committed on a bot's word alone. If you cannot confirm a claim in the code, it is not a fix, however certain the wording.

Then sort each comment into one of two piles:

- **Fix** — verified: a real bug, a security or data-loss risk, a broken edge case, or an unhandled error or null path that you traced in the code.
- **Decline** — wrong (the claim does not hold, or the code already handles it), speculation ("this could theoretically…"), a style preference the repo does not hold, or a suggestion about code this PR never touched.

Never let a bot grow the PR. A fix that lands outside the PR's stated purpose is a decline plus a note to the user.

**3. Act.** Make the fixes as one coherent change, run the repo's own checks (typecheck, lint, tests), commit, and push to the head branch. Then reply to each declined comment with one sentence of reasoning:

```bash
gh api repos/{owner}/{repo}/pulls/<n>/comments/<comment-id>/replies -f body='<reasoning>'
```

A decline you never voiced reads as an oversight, and the bot will raise it again.

**4. Wait.** A push triggers re-review, and bots take a few minutes to answer:

```bash
sleep 180
```

Go back to step 1.

## Going quiet

The PR is quiet, and you are done, when a full cycle passes with no new bot comments after your most recent push. Confirm with `gh pr checks <n>` and report.

Stop early and hand back to the user when:

- A bot re-raises something you already declined. State the disagreement; do not argue it twice.
- A fix needs a decision that is not yours: an API change, a new dependency, or a change in behaviour.
- Checks fail for a reason your changes cannot reach.
- Five rounds pass without the PR going quiet.

Never merge the PR, never force-push, and never touch a branch other than the PR's head.

## Report

Close with: rounds spent, what you fixed and the evidence for each, what you declined and why, and the current state of the PR.
