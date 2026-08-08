# Block markup

Everything the debrief page is built from. Copy a block, fill it, drop it at `<!-- CONTENT -->`.

Use only the classes on this page. The template already styles every one of them.

Top-level sections must be `<section id="…">`; the sidebar builds itself from them. `data-toc` sets the sidebar label when the heading is too long.

---

## hero

Opens the page.

```html
<section id="top" data-toc="Overview" class="hero">
  <h1>Rate limiting on the public API</h1>
  <p class="lede">You asked for a limit on how often anyone can call the API. I added a counter that
  sits in front of every request and turns people away after 100 calls a minute.</p>
  <div class="meta">
    <span class="stat"><b>4</b> work items</span>
    <span class="stat"><b>11</b> files</span>
    <span class="stat"><b>+382</b> / <b>−47</b></span>
    <span class="stat"><b>6 min</b> read</span>
  </div>
</section>
```

## tldr

One paragraph, the whole session. No lists. Someone who reads only this must still know what changed.

```html
<section id="tldr"><h2>In short</h2>
  <p>Three things changed. The API now refuses callers who go over 100 requests a minute. A crash on
  logout is fixed; it happened because the code cleared the session before it read the user's ID.
  And the login page loads faster because it no longer waits for the avatar image.</p>
</section>
```

## file-map

Every file touched, one plain sentence each. Drop it when the session changed no files.

```html
<section id="files"><h2>Files that changed</h2>
  <div class="files">
    <div class="file">
      <div class="file-path">src/middleware/rate-limit.ts</div>
      <div class="file-stat"><span class="add">+120</span> <span class="del">−0</span></div>
      <div class="file-note">New. Counts requests per API key and rejects the ones over the limit.</div>
    </div>
  </div>
</section>
```

## work item

One `<section>` per piece of work. Tag is one of `bug`, `feature`, `improvement`, `research`.

```html
<section id="item-rate-limit"><h2>Rate limiting</h2>
  <div class="item">
    <span class="tag tag-feature">Feature</span>
    <p>…prose…</p>
    <!-- blocks this type needs -->
  </div>
</section>
```

## root-cause

Three headings, in this order, no exceptions.

```html
<h3>What went wrong</h3>
<p>Logging out crashed the app. The screen went blank and the console showed
<code>Cannot read property 'id' of null</code>.</p>
<h3>Why it happened</h3>
<p><code>logout()</code> set the session to <code>null</code>, then called <code>trackEvent()</code>.
That second function reads <code>session.user.id</code>. By the time it ran, there was no session
left to read.</p>
<h3>What fixed it</h3>
<p>The user's ID is now copied into a local variable before the session is cleared.</p>
```

## walkthrough

The path through the new code, start to end, in the order it runs.

```html
<h3>How it works now</h3>
<ul class="plain">
  <li>A request arrives and the middleware reads its API key.</li>
  <li>Redis holds a counter for that key. The middleware adds one to it.</li>
  <li>Over 100 in the last minute, the request is refused with a <code>429</code> status.</li>
</ul>
```

## diff

Trim to the lines that matter, six to twenty. `open` the block only when it is the point of the item.

```html
<details class="diff"><summary>src/auth/logout.ts</summary><pre><span class="ctx">export function logout() {</span>
<span class="del">-  session.clear();</span>
<span class="del">-  trackEvent("logout", { userId: session.user.id });</span>
<span class="add">+  const userId = session.user.id;</span>
<span class="add">+  session.clear();</span>
<span class="add">+  trackEvent("logout", { userId });</span>
<span class="ctx">}</span></pre></details>
```

Escape `<` as `&lt;` inside `<pre>`.

## diagram

Three shapes. Pick the single one that fits the item.

**flow** — a path through steps. Add `vertical` for long chains.

```html
<div class="dg">
  <div class="dg-flow">
    <div class="dg-node">Request<span class="sub">POST /api</span></div>
    <div class="dg-arrow">→</div>
    <div class="dg-node hi">Rate limiter<span class="sub">rate-limit.ts</span></div>
    <div class="dg-arrow">→<span>under limit</span></div>
    <div class="dg-node good">Handler</div>
  </div>
  <div class="dg-caption">Every request now passes the limiter first.</div>
</div>
```

**layers** — what sits on what.

```html
<div class="dg">
  <div class="dg-layers">
    <div class="dg-layer"><div class="name">HTTP layer</div>
      <div class="parts"><span>routes/</span><span>middleware/</span></div></div>
    <div class="dg-layer"><div class="name">Storage</div>
      <div class="parts"><span>redis</span><span>postgres</span></div></div>
  </div>
  <div class="dg-caption">The limiter lives in the HTTP layer and talks only to Redis.</div>
</div>
```

**tree** — file or component structure.

```html
<div class="dg">
  <ul class="dg-tree">
    <li>src/<ul>
      <li>middleware/<ul>
        <li>rate-limit.ts <span class="note">new</span></li>
      </ul></li>
    </ul></li>
  </ul>
</div>
```

## callout

Judgement, not fact. Title is `Why this way`, `Watch out`, or `What I did not do`. Class is
`callout-why`, `callout-gotcha`, or `callout-rejected`.

```html
<div class="callout callout-rejected">
  <div class="callout-title">What I did not do</div>
  <p>I did not put the counter in memory. That works on one server, but you run three, so each one
  would count separately and let three times the traffic through.</p>
</div>
```

## findings

Every claim carries an inline citation that links out.

```html
<h3>What I found</h3>
<p>Redis sorted sets are the usual way to build a sliding window, because they let you drop old
timestamps in one call.<a class="cite" href="https://redis.io/glossary/rate-limiting/" target="_blank" rel="noreferrer">[1]</a></p>
<h3>Sources</h3>
<ol class="sources plain">
  <li>[1] <a href="https://redis.io/glossary/rate-limiting/" target="_blank" rel="noreferrer">Redis, Rate limiting patterns</a></li>
</ol>
```

## timeline

Work abandoned, direction reversed, an approach tried and dropped.

```html
<section id="timeline"><h2>How the session went</h2>
  <ul class="tl">
    <li><strong>First attempt: in-memory counter</strong>
      <span>Dropped once we remembered you run three servers.</span></li>
    <li><strong>Moved to Redis</strong>
      <span>One shared counter that every server sees.</span></li>
  </ul>
</section>
```

## glossary

Words specific to this codebase or domain, glossed for someone who has not read the code.

```html
<section id="glossary"><h2>Words used here</h2>
  <dl class="gloss">
    <dt>tenant</dt><dd>One customer company. All their users share one tenant.</dd>
  </dl>
</section>
```

## check-yourself

What to verify by hand, in the order you would do it. Every entry is something that could plausibly
still be broken.

```html
<section id="check"><h2>Check this yourself</h2>
  <ul class="plain check">
    <li>Call the API 101 times in a minute. The last call should come back <code>429</code>.</li>
    <li>Log out. The app should return to the login screen with no blank flash.</li>
  </ul>
</section>
```

## not-covered

Every gap, named plainly. There is always at least one; find it and write it.

```html
<section id="gaps"><h2>What this page does not cover</h2>
  <ul class="plain">
    <li>Three small files are listed above but not explained. They are one-line import changes.</li>
    <li>I am not sure why <code>retry.ts</code> was touched. It was changed early in the session and
    the reason is not in the diff.</li>
  </ul>
  <footer>Written from the git diff and the files as they now stand.</footer>
</section>
```
