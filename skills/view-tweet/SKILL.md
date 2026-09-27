---
name: view-tweet
description: Read X/Twitter post links as context, exact text, full thread, metrics, quoted post, and media via the fxtwitter API, with the Grok CLI for mid-thread links and replies. Use when the user shares an x.com or twitter.com post link.
---

# View Tweet

Your own fetch tools cannot read X: x.com answers with a 402 or a login wall. Two routes can:

- **fxtwitter API**: free, instant, exact. X's own data as JSON: every post of the author's thread verbatim, metrics, quoted post, media URLs. Blind to reply text.
- **Grok CLI**: Grok's `WebFetch` reads the rendered x.com page. It sees replies and can place a mid-thread post in its thread, but costs Grok usage, takes about 20s, and its output is a model transcription, so long posts arrive cut at "Show more".

fxtwitter is the source of truth; grok fills its gaps. Process each link through the steps below, then use the result as context for whatever the user actually asked.

## 1. fxtwitter

Take the status id from the link (the digits after `/status/`; `twitter.com` and `?s=` query strings are fine):

```bash
curl -s https://api.fxtwitter.com/2/thread/<id> | jq '{code, thread: [.thread[]? | {url, author: .author.screen_name, created_at, text, likes, reposts, replies, views, quote: (.quote | if . then "@\(.author.screen_name): \(.text)" else null end), media: [.media.all[]? | "\(.type) \(.url)"]}]}'
```

Runs fine inside the sandbox. Branch on the result:

- **`thread` has posts**: done. A standalone post is a thread of one. The posts are in order, and each `text` is verbatim.
- **`code` 200 with an empty `thread`**: the link points into the middle of a thread, and fxtwitter returns nothing for it. Go to step 2.
- **`code` 404**: the post is deleted, private, or the link is wrong. Tell the user which link failed. The post's content comes only from a successful read, never from memory or the URL slug.
- **Any other failure** (no JSON, 5xx, timeout): read the post with grok's full read (step 3) instead.

To see an image, download its media URL to `$TMPDIR` and view the file.

## 2. Mid-thread link: find the root

Ask grok for the thread's first post, then rerun step 1 with that id:

```bash
grok -p "$(cat <<'EOF'
Read this X post with WebFetch: <URL>

If it belongs to a thread by the same author, reply with only the x.com status URL of the thread's first post. If it is not part of a thread, reply with only: STANDALONE. If the page does not contain the post, reply with only: NOT FOUND.
EOF
)" --effort low --tools WebFetch
```

On `STANDALONE` (usually a reply in someone else's conversation), use grok's full read in step 3 for the post.

## 3. Replies, or fxtwitter down: grok's full read

Run this when the user's request turns on how people reacted, or when step 1 failed outright:

```bash
grok -p "$(cat <<'EOF'
Read this X post with WebFetch: <URL>

Return, in this order:
- Author: display name and @handle
- Date
- Text: the exact post text, verbatim. If it is a thread by the same author, every post in order, each verbatim, numbered.
- Quoted post: author and verbatim text, if any
- Media: one line per image or video describing it, if any
- Links: expanded URLs in the post
- Replies: up to 3 notable replies, handle plus verbatim text, if the page shows them

Quote, never paraphrase. If the page does not contain the post, reply exactly: NOT FOUND, followed by what the page showed.
EOF
)" --effort low --tools WebFetch
```

Whenever fxtwitter already returned the text, keep its version and take only the replies from grok.

## Running grok

- **Run it outside the sandbox.** Claude Code parent: set `dangerouslyDisableSandbox: true` on the first attempt. Grok writes session files under `~/.grok/sessions`, so a sandboxed run dies at startup with `Couldn't create session: Permission denied ... FS_PERMISSION_DENIED`.
- **`--tools WebFetch` is load-bearing.** It pins grok to the one tool the job needs. Unpinned, grok goes hunting for search skills and shells out, headless mode cancels those shell calls, and the run ends with nothing. The name is exactly `WebFetch`: `web_fetch` silently strips the fetch tool and every link comes back NOT FOUND.
- `--effort low` suffices, this is transcription. Each call costs $0.03 to $0.07 of Grok usage.
- On an auth error, ask the user to run `! grok login`. On quota errors, tell the user replies and mid-thread context are unavailable and continue with what fxtwitter returned.
- Several links needing grok: one call per link, run in parallel with `&`, each redirected to its own `/tmp/tweet-<id>.txt`, then `wait` and read the files.
