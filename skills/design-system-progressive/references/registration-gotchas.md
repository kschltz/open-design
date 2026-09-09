# Registration & tooling gotchas — verified on this codebase (2026-09)

Every item below was hit in practice. Check the whole list before declaring a
design system "registered and done".

## 1. The import scanner is generic — author like it expects

`od design-systems import-local <path>` (POST `/api/design-systems/import/local`)
scans for `tokens.css`, `README.md`, component files, CSS custom properties.
It does **not** read a bespoke DESIGN.md for values. Consequences:

- No `tokens.css` → `tokens.css` is generated with a **neutral fallback
  palette** (`--fg: #767676`, `--accent: #2563eb`, …) and the generated
  `DESIGN.md` says "No CSS custom properties were found". This both empties
  the system and can *contradict* its rules (767676 on white ≈ 4.5:1).
- The generated `DESIGN.md` **overwrites** whatever DESIGN.md existed in the
  source folder. Put the real content into the system *after* import, or pass
  through `import.ts` expectations.
- Import refuses paths inside daemon runtime data (`.od/…`): "cannot import
  OpenDesign runtime data". Stage the source in a normal directory first.
- Extra files next to html imports: `.artifact.json` sidecars are copied too —
  delete them from the staging copy.

## 2. Publish before attach

Projects validate `designSystemId` with `isProjectUsableDesignSystem`:
`status === 'draft'` → `DESIGN_SYSTEM_NOT_PUBLISHED` ("draft design systems
cannot be used by projects"). Newly imported systems are drafts.

```
PATCH /api/design-systems/user:<id>   {"status": "published"}
```

## 3. Two copies exist; the API serves the mirror project

- `.od/design-systems/<id>/` — the design-system directory (materialized from
  the mirror; the token-contract rebuild writes previews here).
- `.od/projects/ds-<id>/` — the **mirror project**. `GET /api/design-systems/:id`
  reads its body via `readDesignSystemWorkspaceTextFile` from this project.

Editing only `.od/design-systems/<id>/` does **not** update what the API (and
therefore the Studio UI) serves, and the mirror only re-materializes on
certain events. After any change: write both copies, then verify the API body.

## 4. The kit UI parser contract (`apps/web/src/runtime/design-md-parse.ts`)

`parseDesignMd` splits on `## ` headings and matches **English keywords** in
the lowercased title (numeric prefixes are stripped). Modules and keywords:

| Module        | Title keywords |
|---------------|----------------|
| Identity      | `identity`, `overview`, `brand` (+ preamble) |
| Typography    | `typograph`, `type`, `font` |
| Color Palette | `color`, `palette` |
| Voice & Tone  | `voice`, `tone`, `messaging` |
| Imagery/Layout| `imagery`, `image`, `photograph`, `illustration`, `layout`, `spacing`, `grid`, `composition` |
| Design System | `design system`, `component`, `token`, `motion`, `interaction` |

Bilingual headings work (`## 2. Color Palette — Tokens de cor`); Portuguese-only
headings render as an empty/blank module. Color rows are read from lines
containing a hex inside the palette-matched section:

- Table row: `| Papel | Nome | Hex | Uso |` → role=col1, name=col2, usage=col4.
- Bullet: `- **Nome:** `#hex` — uso`.
- First hex on the line wins; duplicates deduped by hex (first wins), cap 12.
- A bold fragment before the hex becomes the label — bolding a stray word
  (e.g. `**e**`) names the swatch "e". Put the label bold at line start.
- Verification table of pairs inside the palette section is safe **if** the
  main table comes first (hexes dedupe).

Typography is read from bullets: `- **Display:** Family ("Quoted") — weights 400, 700`.

## 5. Verification snippets (run after every write)

Parser check:

```bash
npx tsx -e "
import { parseDesignMd } from './apps/web/src/runtime/design-md-parse';
import { readFileSync } from 'node:fs';
const p = parseDesignMd(readFileSync('.od/projects/ds-<id>/DESIGN.md', 'utf8'));
console.log(p.name, p.colors.length, p.typography.body?.family);
"
```

Serving check (body actually updated, status actually published):

```bash
curl -s http://127.0.0.1:<port>/api/design-systems/user:<id> | python3 -c "
import json,sys; d=json.load(sys.stdin); ds=d.get('designSystem') or d
b=ds.get('body') or ''
print(ds.get('status'), len(b), '<new-section-marker>' in b)
"
```

Daemon port changes across `tools-dev` restarts — read it from
`pnpm tools-dev status`, never hardcode it.

## 6. Generation runs that extend a system

- Inner agents must **write files whole** (full `write`, no partial edit
  patches) and save each file as soon as it is done. A failed edit call
  terminated an entire run with zero artifacts.
- Runs inherit the project's `designSystemId` automatically
  (`designSystemSelectionSource: "project"`), so keep the attachment valid.
- A run ending `succeeded` with `artifactCount: 0` means the agent said "I'll
  now do X" and died — check `agentMessage` in `get_run` before assuming
  success, and relaunch with the whole-file rule stated in the prompt.