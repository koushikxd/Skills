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

**YAML frontmatter = the tokens** (machine-readable, the literal resolved values), **prose body = how to apply them**. Use the format's canonical groups and section order; drop any the project doesn't support and mark partial data `unknown`. Curly-brace references (`{colors.primary}`) point one token at another.

### Frontmatter (tokens)

Canonical groups the format defines: `colors`, `typography`, `rounded`, `spacing`, `components`. `shadows`, `motion`, and `breakpoints` are extensions — real values the format has no group for; keep them, a consuming agent still reads them, but they're not part of the standard groups.

```yaml
---
name: <design system / product name>
description: <one line; note theme if this file is one of several, e.g. Light>
colors:
  # flat, semantic-first names; literal resolved values (hex / oklch / rgba).
  # name tokens so the binding is derivable — `background-primary` → `bg-background-primary`
  primary: "#171717"
  background: "#ffffff"
  accent: "#0070f3"   # inferred from 40 usages   ← annotate inferred; measured tokens need no note
  # ramps where they exist: gray-100 … gray-1000, plus accent scales
typography:
  # named scales; each maps only canonical fields:
  # fontFamily, fontSize, fontWeight, lineHeight, letterSpacing
  heading-32: { fontFamily: …, fontSize: 32px, fontWeight: 600, lineHeight: 40px, letterSpacing: -1.28px }
  body-16:    { fontFamily: …, fontSize: 16px, fontWeight: 400, lineHeight: 24px }
rounded:      { sm: 6px, md: 12px, lg: 16px, full: 9999px }   # if source has only --radius, write { base: … }
spacing:      { base: 4px, 1: 4px, 2: 8px, … }   # the steps actually used
components:
  button-primary: { backgroundColor: "{colors.primary}", textColor: "…", rounded: "{rounded.sm}", height: 40px }
  input:          { … }
# --- extensions (not spec groups) ---
shadows:      { card: "…", popover: "…", modal: "…" }
motion:       { easing: "cubic-bezier(…)", fast: 150ms, base: 200ms }
breakpoints:  { sm: …, md: …, lg: …, xl: … }
---
```

Shape rules that keep the frontmatter standard-readable:
- `colors`, `rounded`, `spacing` are **flat** maps of `token-name: value`. Names match `^[a-zA-Z0-9][a-zA-Z0-9-]*$` — `background-primary`, never nested `background.primary`.
- `typography` is a map of **named scales**; each scale uses only the canonical fields above. Never a bare scalar (`sans: Geist`) or renamed keys (`family`/`size`).
- Don't invent scale steps the source lacks — one `--radius` becomes `rounded.base`, not a fabricated `sm`/`md`/`lg`.

Inside `components`, fix the conventions, not the shape. A button's fields aren't an input's, so don't force a uniform set — but hold two conventions: (1) every value uses the `{colors.x}` / `{rounded.y}` reference style, never a raw literal, so its source stays visible; (2) states use the fixed key names `hover` / `active` / `disabled` / `focus`. This block earns its detail mainly for **un-tokenized** projects. When the project ships real primitives, the consumer should `import <Button variant="primary">`, not rebuild from this spec — keep the entry thin and let the Consumption section carry the import path.

### Prose sections

Canonical order, extensions slotted in place. Provenance rules are in SKILL Step 2 — don't ship `file:line`; the binding is the citation.

- **Overview** *(canonical)* — open with the **atmosphere**: 2–3 sentences of evocative mood (minimal/high-contrast? dense/airy? warm/clinical?), read *from* the grounded tokens, never invented. Then state which theme this file documents, and note if any section is inferred from usage rather than a token system.
- **Consumption** *(extension, placed right after Overview)* — the binding mechanism (Tailwind utilities / CSS vars / theme object / component imports), the import path for shared primitives, and the rule to reach for the binding over the literal.
- **Colors** *(canonical)* — what each scale step means (e.g. `100` bg → `1000` text), which surfaces vs text vs borders, what each accent signals (error/warning/success/link), light vs dark.
- **Typography** *(canonical)* — the families and their roles (UI / prose / code), and which styles cover most text.
- **Layout** *(canonical)* — the spacing rhythm, container width, grid, breakpoints, mobile behavior.
- **Elevation & Depth** *(canonical)* — the real shadow values and when each applies, each with its physical read ("whisper-soft", "heavy drop"); whether hierarchy leans on shadows or on tonal surfaces/borders.
- **Motion** *(extension)* — durations, easing, and the rule for when motion is used; honor `prefers-reduced-motion`.
- **Shapes** *(canonical)* — the radius scale and which radius goes on which surface. Pair each value with its physical read (`9999px` → "pill", `12px` → "softly rounded", `0` → "sharp").
- **Components** *(canonical)* — per-primitive specs (sizes, states: hover/active/disabled/focus), drawn from the real component code.
- **Patterns** *(extension)* — recurring composition recipes: page scaffolding, form/list/table/empty-state/header layouts, the spacing rhythm between sections. The layout grammar that makes a new feature read as native, beyond raw tokens. Drawn from real screens; omit if the project has too few to generalize.
- **Voice & Content** *(extension)* — casing, action-label patterns, error/toast/empty-state conventions, taken from real strings.
- **Do's and Don'ts** *(canonical)* — the project's actual conventions stated as rules ("reference tokens, never raw hex"; "don't mix rounded and sharp corners").
