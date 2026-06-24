# Reference

Two parts: **A.** where each token type lives, per stack (for Step 1). **B.** the DESIGN.md anatomy to produce (for Step 3).

---

## A. Where tokens hide, by stack

Search these first; follow every reference to its literal value.

**Tailwind (v3 & v4)**
- `tailwind.config.{js,ts}` → `theme.extend` for colors, spacing, `borderRadius`, `boxShadow`, `fontFamily`, `screens` (breakpoints), `keyframes`/`animation`.
- Tailwind v4: `@theme` block and `@theme inline` in `globals.css`/`app.css` — tokens defined as CSS vars and exposed as utilities.
- Class usage in components reveals the *de-facto* scale even when the config is sparse.

**Plain CSS / CSS variables**
- `:root` and `.dark` (or `[data-theme]`) blocks in `globals.css`, `app.css`, `index.css`, `theme.css` → custom properties for color, spacing, radius, shadow, easing, duration.
- `@font-face` and `@import` for fonts; `@media (min-width:…)` for breakpoints; `@keyframes` for motion.

**CSS-in-JS / component frameworks**
- styled-components / Emotion: a `theme.ts`/`theme.js` object passed to `ThemeProvider`.
- MUI: `createTheme({...})` — `palette`, `typography`, `spacing`, `shape.borderRadius`, `shadows`.
- Chakra: `extendTheme({...})` — `colors`, `fonts`, `space`, `radii`, `shadows`.

**Design-token systems**
- `tokens.json`, `design-tokens.json`, Style Dictionary configs, `*.tokens` — the source of truth when present; prefer these over derived values.

**Mobile / native**
- SwiftUI: asset catalog `*.colorset` JSON, `Color(...)`/`Font(...)` extensions, a `Theme.swift`.
- Flutter: `ThemeData(...)`, `ColorScheme`, `TextTheme` in `main.dart`/`theme.dart`.
- React Native: `StyleSheet.create`, a shared `theme`/`tokens` module.

**Cross-cutting**
- Bindings: while reading the sources above, note *how* components reference tokens — Tailwind utility classes, `var(--…)`, a `theme.x` path, or component imports. That mechanism is the Consumption section; capture it, not just the values.
- Fonts: `next/font` calls, `@font-face`, `<link>` to font CDNs, `fontFamily` in config.
- Components: the UI primitives dir (`components/ui`, `src/components`, a package) — read variants (`cva`/`class-variance-authority`, prop-driven styles) for button/input/card specs and state (hover/active/disabled/focus) handling.
- Composition: the page/feature/route dirs (`app/`, `pages/`, `src/features`) — read a few real screens for layout recipes: page scaffolding, form/list/table/empty-state structure, section spacing rhythm. This is the Patterns section.
- Voice & content: landing page, marketing copy, button labels, toasts, empty states, error messages — extract tone, casing, and conventions from real strings.
- Themes: detect light/dark (or more) and capture each set of values separately.

---

## B. DESIGN.md anatomy

Mirror the structure of the Geist / Mintlify examples: **YAML frontmatter = the tokens** (machine-readable, the literal resolved values), **prose body = how to apply them**. Drop any section the project doesn't support; mark partial data `unknown`.

### Frontmatter (tokens)

```yaml
---
name: <design system / product name>
description: <one line; note theme if this file is one of several, e.g. Light>
consumption:
  # how this codebase binds tokens to code — what an author types
  tokens: tailwind        # tailwind | css-vars | theme-object | inline
  example: "bg-background-primary, text-foreground-secondary"   # real bindings, not hex
  components: "@/components/ui"   # import primitives from here; don't rebuild them
  rule: "reference the binding, never a raw literal"
colors:
  # semantic first, then scales; literal resolved values (hex / oklch / rgba)
  primary: "#171717"
  background: "#ffffff"
  accent: "#0070f3"   # inferred from 40 usages   ← annotate inferred values; tokenized ones need no citation
  # ramps where they exist: gray-100 … gray-1000, plus accent scales
typography:
  # one entry per real text style, with the metrics the project actually sets
  heading-32: { fontFamily: …, fontSize: 32px, fontWeight: 600, lineHeight: 40px, letterSpacing: -1.28px }
  body-16:    { fontFamily: …, fontSize: 16px, fontWeight: 400, lineHeight: 24px }
spacing:      { base: 4px, 1: 4px, 2: 8px, … }   # the steps actually used
rounded:      { sm: 6px, md: 12px, lg: 16px, full: 9999px }
shadows:      { card: "…", popover: "…", modal: "…" }
motion:       { easing: "cubic-bezier(…)", fast: 150ms, base: 200ms }
breakpoints:  { sm: …, md: …, lg: …, xl: … }
components:
  button-primary: { backgroundColor: "{colors.primary}", textColor: "…", rounded: "{rounded.sm}", height: 40px }
  input:          { … }
---
```

Fix the conventions, not the shape, inside `components`. A button's fields aren't an input's aren't a card's, so don't force a uniform field set — but hold two conventions so the block stays predictable across runs: (1) every value uses the `{colors.x}` / `{rounded.y}` reference style, never a raw literal, so its source stays visible exactly as Geist does; (2) states use the fixed key names `hover` / `active` / `disabled` / `focus`. This block earns its detail mainly for **un-tokenized** projects, where the spec is the only source. When the project ships real primitives, the consumer should `import <Button variant="primary">`, not rebuild from this spec — so keep the entry thin and let `consumption.components` carry the import path.

Provenance in the output: tokenized values need no citation — their binding names where they live. Annotate **inferred** values inline with `# inferred from N usages` so the reader can tell a measured token from a derived one. Don't ship `file:line` citations.

### Prose sections

- **Overview** — open with the **atmosphere**: 2–3 sentences of evocative mood (minimal/high-contrast? dense/airy? warm/clinical?), read *from* the grounded tokens, never invented. Then state which theme this file documents, and note if any section is inferred from usage rather than a token system.
- **Consumption** — how an agent applies these tokens in *this* codebase. Name the binding mechanism (Tailwind utilities / CSS vars / theme object / component imports), the import path for shared primitives, and the hard rule: reference the binding, never a raw literal. This is the section that turns the spec into correct code instead of look-alike code — put it second, right after Overview.
- **Colors** — what each scale step means (e.g. `100` bg → `1000` text), which surfaces vs text vs borders, what each accent signals (error/warning/success/link), light vs dark.
- **Typography** — the families and their roles (UI / prose / code), and which styles cover most text.
- **Layout** — the spacing rhythm, container width, grid, breakpoints, mobile behavior.
- **Elevation & Depth** — the real shadow values and when each applies, each with its physical read ("whisper-soft", "heavy drop"); whether hierarchy leans on shadows or on tonal surfaces/borders.
- **Motion** — durations, easing, and the rule for when motion is used; honor `prefers-reduced-motion`.
- **Shapes** — the radius scale and which radius goes on which surface. Pair each value with its physical read (`9999px` → "pill", `12px` → "softly rounded", `0` → "sharp").
- **Components** — per-primitive specs (sizes, states: hover/active/disabled/focus), drawn from the real component code.
- **Patterns** — recurring composition recipes: page scaffolding, form/list/table/empty-state/header layouts, the spacing rhythm between sections. This is the layout grammar that makes a new feature read as native, beyond raw tokens. Drawn from real screens; omit if the project has too few to generalize.
- **Voice & Content** — casing, action-label patterns, error/toast/empty-state conventions, taken from real strings.
- **Do's and Don'ts** — the project's actual conventions stated as rules ("reference tokens, never raw hex"; "don't mix rounded and sharp corners").
