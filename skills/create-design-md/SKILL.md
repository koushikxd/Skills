---
name: create-design-md
description: Generate or update a DESIGN.md — a grounded design-system spec extracted from a project's real tokens, components, and layout patterns so other agents build new UI that matches the app. Use when the user wants to create or update a DESIGN.md, or teach an agent the app's design language to build new features in the same style.
---

# Capture a Project's Design System → DESIGN.md

You read a project's UI code and distill it into a `DESIGN.md` that another agent can follow to build new screens indistinguishable from the existing ones — like the curated `DESIGN.md` files Vercel (Geist) and Mintlify ship.

Those examples are curated *to their own apps*. Yours must be too. The document is only useful if every value is **ground truth** — pulled from this project's code, not invented. A plausible palette that isn't the project's real palette is worse than a gap: it produces confidently-wrong UI that looks intentional.

One rule sits under everything: **every token traces to a source.** Follow it everywhere. When you can't find a value, you mark it `unknown` — you never fill it with a sensible default.

## Step 1 — Scope the run

First, is this a fresh spec or an update? If `DESIGN.md` already exists, this is an **update** — the expected case as a project grows past the thin spec it was bootstrapped with. Read the existing file; you'll reconcile against it while extracting in Step 2.

Then find where design decisions live. Auto-detect the token homes — theme config, CSS variables, font setup, the component library, page/feature dirs, brand/marketing copy. `REFERENCE.md` has the per-stack cheatsheet of where each hides (Tailwind, CSS vars, CSS-in-JS / MUI / Chakra, design-token JSON, SwiftUI, Flutter, React Native) — read it now.

Real apps are inconsistent — current design, legacy screens, a half-migrated middle — so don't average them blindly. Propose the detected scope and let the user redirect: *"I found your theme in X and primitives in Y — I'll base the spec on these. Any canonical pages to focus on, or legacy areas to exclude?"* Skip the question only when the project is small enough to have one coherent style.

Branch on what you found — three regimes:
- **Existing UI** (tokenized or not) → extract in Step 2; for un-tokenized projects infer the de-facto system by frequency and label it inferred.
- **Theme/config only** (freshly-init'd: a palette, a `tailwind.config`, a `shadcn` base, but no feature UI) → extract straight from the config. This is real ground truth, not fabrication; the spec is thin and unknowns stay `unknown`. Expect to re-run this as the project grows.
- **Truly empty** (no design decisions at all) → nothing to ground a spec in. Interview the user for a brand brief, or stop — never fabricate one.

**Completion criterion:** the run is marked new or update, the scope is confirmed, and a source map names the file(s) for each of — colors, typography, spacing, radii, shadows, motion, components, layout patterns, voice. A category with no source is recorded as "none found", not skipped silently.

## Step 2 — Extract, don't invent

Pull the real values from the sources you mapped. If Step 1 flagged this as an **update**, reconcile rather than rewrite: refresh changed values, fill former `unknown`s now that real UI exists, drop tokens whose source is gone, and keep curated prose that still holds. Hold the ground-truth rule:

- **Capture the binding, not just the value.** A value is useless to a coding agent if it doesn't know how this codebase *consumes* it. For each token record the **binding** — the symbol an author actually types: the Tailwind utility (`bg-background-primary`), the CSS var (`var(--primary)`), the theme path (`theme.colors.primary`), or the imported component (`<Button variant="primary">`). The literal value is the fallback for new tokens; the binding is what the agent must use for existing ones. An agent that writes raw `#171717` instead of the binding produces UI that looks right but breaks on theme changes and bypasses the design system.
- **Resolve indirection to the literal.** A Tailwind color → CSS var → hex chain gets followed to the end; record both the binding *and* the resolved value (the Geist example references `{colors.primary}` while the frontmatter holds the literal hex). Capture light *and* dark values separately when the project themes (Mintlify lists light → dark).
- **Reusable components are the strongest binding.** Where the project ships UI primitives (`components/ui`, a component package), the match-the-app instruction is "import the real `<Button>`", not "rebuild a button from specs". Record the import path and the variant props, not just the visual spec.
- **Read usage to capture intent, not just the value.** Which step is hover vs active, which color means error, which radius is for cards vs inputs. A value without its role can't be applied correctly.
- **Un-tokenized → frequency.** The hex used in 40 places is `primary`; the three radii actually used become the scale. Derive the system from what the code does, and label it inferred.
- **Gaps stay gaps.** A category your source map marked "none found" is written `unknown` in the output. Never substitute a default from the example files or from memory.

**Where provenance lives in the output.** Don't litter the shipped file with `file:line` citations — they serve you, not the consumer, who wants the value and the binding, not its address. For tokenized values the **binding *is* the citation**: `bg-background-primary` already says where the value lives, more usefully than `colors.ts:22`. Reserve an explicit note for **inferred** values only — they have no binding, so they carry an inline `# inferred from N usages` so the reader can tell a measured token from a derived one. Keep raw `file:line` references in your working notes; you'll need them for the Step 4 audit and for update runs, but they don't ship.

**Completion criterion:** every extracted value is traceable — tokenized values via their binding, inferred values via an `inferred from N usages` annotation; every "none found" category is marked `unknown`. No value is present that you couldn't point at in the codebase.

## Step 3 — Synthesize DESIGN.md

Write `DESIGN.md` at the project root following the anatomy in `REFERENCE.md`: machine-readable YAML frontmatter (the tokens) followed by prose sections that explain *how* to apply them — Overview, **Consumption**, Colors, Typography, Layout, Elevation, Motion, Shapes, Components, **Patterns**, Voice & Content, Do's and Don'ts. Read `REFERENCE.md` for the full template before writing.

The **Consumption** section is what makes the file produce correct code rather than look-alike code: state plainly how this project binds tokens to code — "reference tokens via Tailwind utilities (`bg-background-primary`), never raw hex"; "import primitives from `@/components/ui`" — so an agent building a new feature reaches for the binding, not the literal value.

The **Patterns** section is what lets the file stand alone instead of sending an agent to study a sibling feature: capture the app's recurring composition — page scaffolding, how forms/lists/empty-states/headers are assembled, the spacing rhythm between sections. Tokens give the look; patterns give the layout grammar that makes a new feature read as native. Omit it only when the project has too few screens to generalize (a thin bootstrap spec); it fills in on the update run.

Tokens tell an agent the exact values; they don't tell it the *feel* that lets it extrapolate to a screen the spec never covered. Capture both:
- **Atmosphere.** Open the Overview with 2–3 sentences of evocative mood — "airy and high-contrast," "dense and utilitarian," "warm editorial." This is a *reading of the grounded tokens* (large spacing → airy; tight contrast → calm), never free invention, so it stays true to the project.
- **Physical descriptors.** Alongside raw values, give the one-word physical read: `9999px` → "pill", `6px` → "barely-rounded", `0 1px 2px` → "whisper-soft". The agent gets precision *and* intuition.

Write only sections your extraction supports. Inferred or `unknown` material is labeled as such in place — the reader must be able to tell a measured token from a guessed one.

Keep it one file and keep it lean. This is the file an agent loads whole to place one button, so it competes for context every time. The frontmatter holds values; prose holds *application* — "`gray-1000` is body text, `gray-500` is muted metadata", not "our typography creates a harmonious reading experience". Anything that is neither a value nor a rule for applying one gets cut. The frequency rule already keeps tokens lean (don't document a 10-step ramp when 4 are used); apply the same restraint to prose. Keep the prose plain, too — no decorative glyphs or symbols dressing up warnings and list items; the agent reads the sentence, not the ornament. A symbol earns a place only when it is itself an extracted value, never as decoration. Don't split into multiple files — the whole value is pointing an agent at one path.

**Completion criterion:** `DESIGN.md` exists with frontmatter tokens and prose; every section is either populated from Step 2 or explicitly marked `unknown`/omitted-with-reason.

## Step 4 — Verify against the code

A spec that drifts from the code is a liability. Audit your own output:
- Confirm each frontmatter token appears in a real source (spot-check the citations from Step 2).
- Confirm the Consumption bindings are real — the utility classes, CSS vars, or import paths you documented must resolve in the codebase. A wrong binding makes the agent write broken code.
- Sample 3–5 actual components and check they're expressible in the documented tokens *via the documented bindings*. A component using a color, radius, or spacing your spec doesn't list means you missed a token — go back and add it, or note it as an intentional exception.

**Completion criterion:** every documented token is confirmed against a source, every binding resolves in the codebase, and the sampled components contain no undocumented values. Report what you sampled and anything that didn't fit.
