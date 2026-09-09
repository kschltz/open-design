---
name: design-system-progressive
description: |
  Create OpenDesign design systems in verified layers instead of one big
  risky bang. Builds the system through three disclosure levels (foundation
  → core components → depth), each ending in a hard verification gate, and
  encodes every registration/parsing gotcha that has historically broken
  user-authored design systems (empty token imports, unpublished drafts,
  mirror-project drift, non-English headings invisible to the kit parser).
  Trigger keywords: "create a design system", "new design system",
  "progressive design system", "design system skill", "registrar design system".
triggers:
  - "create a design system"
  - "new design system"
  - "design system progressive"
  - "register design system"
  - "criar design system"
od:
  mode: design-system
  platform: desktop
  scenario: planning
  design_system:
    requires: false
    generates: true
    sections: [identity, color-palette, typography, component-stylings, layout, voice-tone, accessibility-contract, anti-patterns]
  inputs:
    - name: brief
      type: string
      required: true
      description: "Audience, constraints, and any hard requirements (contrast, language, tone)."
  outputs:
    primary: DESIGN.md
    secondary: tokens.css
    tertiary: components.html, demo/
  capabilities_required:
    - file_write
---

# Design System — Progressive Disclosure Workflow

Build the system in **layers that are each complete and usable**, never as one
giant generation. Each level ends in a **gate**; if the gate fails, fix before
writing the next level. This is the same discipline as UI progressive
disclosure: show the minimum needed to proceed, reveal depth on demand, and
never let depth destabilize the base.

**Iron rule of execution:** write every file **complete and whole** the moment
it is ready. Never splice content into an existing file with partial edits —
a malformed edit loses the whole run's work (this has happened).

## Level 0 — Brief (before any file)

Capture exactly three things, in one round of questions, nothing more:

1. **Audience** — who uses it, education level, vision constraints, language.
2. **Non-negotiables** — e.g. minimum contrast ratio, body size, touch target,
   column width, "state never by color alone".
3. **Scope** — which disclosure levels are wanted now (see below). Default: 1+2.

Freeze these as a short "non-negotiables" block at the top of DESIGN.md. Every
later level must re-read it before writing.

## Level 1 — Foundation (tokens + minimal DESIGN.md)

Produce **only**:

- `tokens.css` — OD contract names (`--bg`, `--fg`, `--accent`, `--border`,
  `--danger`, `--success`, type scale, spacing, radius, focus) with the real
  values and the **measured contrast ratio as an inline comment on every color
  pair**. Aliases for friendly names may follow.
- `DESIGN.md` with: preamble (one-line system summary), `## 1. Identity — …`
  (principles), `## 2. Color Palette — …` (parser table, see reference),
  `## 3. Typography — …` (parser bullets), `## 4. Imagery & Layout — …`
  (spacing/form/grid). Nothing deeper yet.

**Gate 1 (all must pass):**
- [ ] Every color pair ≥ the brief's minimum contrast (compute it, don't eyeball).
- [ ] `tokens.css` uses real values — **never** ship the neutral fallback
      palette (fallback = the system imports as empty).
- [ ] Run the kit-parser check (`npx tsx` snippet in reference) — name,
      colors with roles, typography families all extracted.

## Level 2 — Core components

Add **6–8 essential components** only (button, field, choice card, one-step
nav, error message, success message, plus what the brief demands). Each with
**use and counter-use**, implemented in `components.html` + one demo page
showing a realistic flow.

**Gate 2:**
- [ ] Components reference **only** registered tokens (grep for stray hex).
- [ ] Every state is stated in words, never color alone.
- [ ] Demo page renders without any token that is not in `tokens.css`.

## Level 3 — Depth (only after 1 and 2 pass)

Now and only now: extended components, flow patterns, voice & tone (with a
diga/não-diga table), accessibility contract (pair table, tab order, ARIA,
zoom, touch), anti-patterns gallery, and full demo flows. Extend the existing
files; never rewrite Level 1 tokens at this stage.

**Gate 3:** re-run the Level 1 parser check + contrast audit; confirm the
doc's `##` section count and that no demo uses an unregistered value.

## Registration (when the system must be reusable)

Follow the registration checklist in `references/registration-gotchas.md`
**in order**. The two that bite hardest: the design system must be
**published** before a project can attach it, and the API serves content from
the **mirror project**, so post-registration sync must be verified against the
API body, not the local directory.

## Hard-won rules (never violate)

1. **Author `tokens.css` from the start.** The `import-local` scanner has no
   DESIGN.md reader; a tokens-less project imports with a neutral fallback
   palette that silently contradicts the system.
2. **Headings the kit parser can see.** Section titles must carry the English
   keywords (`Color Palette`, `Typography`, `Voice & Tone`, `Imagery & Layout`,
   `Design System`, `Identity`) — bilingual titles are fine, Portuguese-only
   titles render as an empty kit. Color table must be `| Papel | Nome | Hex | Uso |`.
3. **Publish before attach; verify after sync.** Status `draft` →
   `DESIGN_SYSTEM_NOT_PUBLISHED` on attach. After any file change, fetch the
   API body and confirm the new content is actually being served.
4. **One level at a time, gates between.** A deep system whose foundation
   fails contrast is worse than a small system that passes.