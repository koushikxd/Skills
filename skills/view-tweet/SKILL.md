---
name: view-tweet
description: Read X/Twitter posts through the Grok CLI, returning verbatim text, the full thread, quoted post, media, and top replies as context. Use when the user shares an x.com or twitter.com post link.
---

# View Tweet

Your own fetch tools cannot read X: x.com answers with a 402 or a login wall. Grok's `WebFetch` gets the rendered post, including every post in the author's thread. Run `grok` headless once per link, then use its transcription as context for whatever the user actually asked.

## Invocation

Substitute the link for `<URL>`, keep the rest of the prompt as is:

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

- **Run it outside the sandbox.** Claude Code parent: set `dangerouslyDisableSandbox: true` on the first attempt. Grok writes session files under `~/.grok/sessions`, so a sandboxed run dies at startup with `Couldn't create session: Permission denied ... FS_PERMISSION_DENIED`.
- **`--tools WebFetch` is load-bearing.** It pins grok to the one tool the job needs. Unpinned, grok goes hunting for search skills and shells out, headless mode cancels those shell calls, and the run ends with nothing. The name is exactly `WebFetch`: `web_fetch` silently strips the fetch tool and every link comes back NOT FOUND.
- `--effort low` suffices, this is transcription. Expect about 20s and $0.03 to $0.07 of Grok usage per link.
- `twitter.com` links and links with `?s=` query strings work as given.

**Several links:** one call per link, run in parallel, each to its own file named by status id, then read the files:

```bash
grok -p "..." --effort low --tools WebFetch > /tmp/tweet-2103498682532253734.txt 2>&1 &
grok -p "..." --effort low --tools WebFetch > /tmp/tweet-1519480761749016577.txt 2>&1 &
wait
```

## Using the output

- The transcription is context, not the deliverable. Work from it on the user's request; relay the post itself when they asked what it says.
- `NOT FOUND` means the post is deleted, private, or the link is wrong, and grok includes what the page showed instead. Tell the user which link failed. The post's content comes only from a successful read, never from memory or the URL slug.

## When grok fails

On an auth error, ask the user to run `! grok login`. On quota errors or timeouts, fall back to the fxtwitter API:

```bash
curl -s https://api.fxtwitter.com/<handle>/status/<id> | jq -r '.tweet | "\(.author.screen_name) \(.created_at)\n\(.text)"'
```

It returns only the linked post, never the rest of the thread, quoted replies, or media descriptions, so tell the user the context may be partial.
