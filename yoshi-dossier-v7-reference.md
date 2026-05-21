# Yoshi Dossier — v7.0 Build Reference

**Document:** `index.html` (~4815 lines, single self-contained file)
**Hosted:** https://jalulia.github.io/yoshi
**Repo:** github.com/jalulia/yoshi (GitHub Pages, `main` branch)
**Backend:** Supabase (Postgres + Storage)
**Supabase URL:** `https://odhvdtlhgvrpxcxsuzke.supabase.co`
**Started:** v4 (16 Apr 2026) → v7.0 (21 May 2026)

---

## Architecture

Single HTML file. No build step, no frameworks, no npm. All CSS in one `<style>` block, all JS in five `<script>` blocks at the bottom. Supabase provides persistence via REST API (anon key, no auth). GitHub Pages serves static.

### Script blocks (in order)
1. **DossierApp** — shared Supabase config (`sb()`, `storage()` helpers), version constants
2. **Quick-Jump + Formatting Bar + TOC Manager + Block Insertion** — all UI features
3. **§ 08 Tracker** — standalone task tracker with drag/drop, its own undo stack
4. **Section Editor** — contenteditable per-section, save/cancel, SVG protection

### Supabase tables
| Table | Purpose |
|---|---|
| `dossier_meta` | Single row: version, compiled date, title |
| `dossier_sections` | 9+ rows: section id, sort_order, title, subtitle |
| `dossier_files` | Uploaded file metadata (references Storage paths) |
| `section_content` | Saved section HTML (id → html blob) |
| `tasks` | § 08 tracker items (pre-existing) |

### Supabase Storage
- Bucket: `dossier-files` (public)
- Path pattern: `uploads/{timestamp}-{filename}`
- Public URL: `{SUPABASE_URL}/storage/v1/object/public/dossier-files/{path}`

---

## Features — Complete Inventory

### Navigation
- **Quick-jump menu** — fixed ☰ top-right, slide-out panel from right, lists all sections with § number + title + subtitle. Auto-generated from DOM. Escape to close.
- **Sticky section headers** — always-on (not just edit mode). Section head pins to top of viewport as you scroll.
- **TOC links** — click any § in Contents to smooth-scroll.

### Section Editing
- **Per-section edit** — hover section header → EDIT button appears. Click → section becomes `contentEditable`. SAVE/CANCEL in section header + ADD panel sidebar.
- **Sticky save** — section header sticks to viewport top while editing. If scrolled past, use the Save/Cancel buttons in the ADD panel.
- **Escape to cancel** — restores original HTML from snapshot.
- **Auto-save to Supabase** — `section_content` table, keyed by section id.
- **HTML sanitization** — on save AND on load, strips: `.section-edit-btn`, `.add-point`, `.table-controls`, `.svg-edit-hint`, `.block-delete`, `.fig-upload-zone`. Prevents UI artifacts from being baked into Supabase.

### Inline Formatting
- **Floating toolbar** — appears on text selection inside editing sections. Dark bar with B (bold), I (italic), link icon.
- **Link popover** — click link icon → URL input appears. Enter to apply. Empty URL removes link. Pre-fills existing link href.
- **Keyboard shortcuts** — Ctrl/Cmd+B (bold), Ctrl/Cmd+I (italic), Ctrl/Cmd+K (link), Ctrl/Cmd+Z (undo, browser native).
- **Scope** — only fires inside editing sections, not in inputs, not in § 08 tracker.

### SVG Protection
- **Locked by default** — all `.figure` and `.fig-frame` elements get `contenteditable="false"` on edit enter. CSS `pointer-events: none` on SVGs.
- **Double-click to unlock** — "DOUBLE-CLICK TO EDIT" hint on hover. Double-click → amber outline, figure becomes editable. Click outside → re-locks.
- **Cleaned on save** — SVG editing classes and hint badges stripped.

### TOC Management
- **Manage button** — top-right of Contents section. Click → TOC rows become editable inputs (title + subtitle).
- **Reorder** — ↑/↓ buttons swap TOC rows AND the actual `.page` divs in the DOM.
- **Add section** — creates full scaffolding: `.page` > `.running-head` + `section.section` > `.section-head` + empty prose.
- **Delete section** — removes TOC row + `.page` div + Supabase rows. Confirm dialog (kept for destructive section-level action).
- **Cascade** — title/subtitle changes update: section-name, section-tag, running-head center text, quick-jump menu, Supabase `dossier_sections`.
- **Auto-renumber** — display numbers (§ 01, § 02...) recalculate on any change. Section IDs stay stable.
- **Link navigation killed** — during manage mode, `<a>` clicks are blocked via capture-phase listener.

### Block Insertion (ADD Panel)
- **Persistent sidebar** — 220px, slides from left on edit enter. No grey overlay, doesn't block content. Closes on Save/Cancel/Escape/✕.
- **Section indicator** — shows "Editing — {section name}" with Save/Cancel buttons always accessible.
- **Insertion points** — orange dotted `+` circles between every content block. Default 50% opacity, solid fill on hover. Click to set insertion target (highlighted).
- **No-target fallback** — clicking a block type without selecting a `+` appends at end of section.
- **Refresh after insert** — `injectAddPoints()` re-runs after every insert/delete so new blocks get `+` markers.

#### Block types
| Type | Panel label | What it creates |
|---|---|---|
| `subhead` | § Section heading | `.sub-head` with auto-numbered `.sub-num` + `.sub-name` |
| `prose` | ¶ Body paragraph | `.prose` > `<p>` |
| `heading` | H Subheading | `.sub-head.no-num` > `.sub-name` (full-width, no number) |
| `note` | ※ Note / callout | `.stamp-card[data-stamp="Note"]` > `<h4>` + `<p>` |
| `divider` | — Divider | `<hr>` |
| `image` | ▣ Image / figure | Upload zone (drag/drop + browse + title/desc fields) |
| `table` | ⊞ Table | `.compare` > `<table>` (2×2 default) + row/col controls |
| `kv` | = Key-value pair | `.kv` > `<dl><dt><dd>` |
| `linkcard` | ↗ Link card | `<a class="link-card">` with editable title, desc, URL |

### Block Delete
- **Always visible** — ✕ button on right edge of every content block in edit mode.
- **Two-state click** — first click → turns red "delete?". Second click within 2s → deletes. Auto-resets.
- **Block undo** — snapshot pushed before each insert/delete. Ctrl+Z restores if the last action was a block operation.

### Table Controls
- **Always visible in edit mode** — `+Row`, `+Col`, `−Row`, `−Col` buttons below each table.
- **Minimum constraints** — won't delete below 1 row or 2 columns.

### File Upload
- **Drag/drop zone** — appears when inserting Image/figure block. Accepts images, `.glb`, `.gltf`, `.skp`.
- **Supabase Storage** — full quality, no compression. Files go to `dossier-files` bucket.
- **Metadata** — saved to `dossier_files` table (section_id, filename, storage_path, file_type, title, description).
- **Image rendering** — `<img>` in `.fig-frame` with `.fig-cap` caption.
- **Figure edit** — ✎ button on hover → inline title/description editing with Save.
- **Figure delete** — 🗑 button, two-state click.

### 3D GLB Viewer
- **Lazy-loaded** — Three.js r128 + OrbitControls + GLTFLoader loaded from CDN only when first GLB is dropped.
- **Rendering** — `<canvas>` in `.fig-frame`, auto-center, auto-scale to fill. OrbitControls for rotation.
- **"3D" badge** — top-left corner of the figure frame.
- **Same figure treatment** — caption, edit, delete all work identically to images.

### Print
- `@media print` hides all editing UI, removes sticky/fixed positioning, adds page breaks before sections, clean white backgrounds.

### Mobile/Tablet
- At ≤760px: ADD panel narrows to 200px, quick-jump panel narrows to 260px, edit buttons and add-points compact, formatting bar tightens.

---

## Rules & Patterns

### Version Bumping
- Each build increments: v7.0, v7.1, v7.2...
- **4 locations to update:** cover header, cover meta, colophon, JS `VERSION` constant.
- **Use context-aware sed** — match surrounding text (e.g., `"Project Dossier · v7.0"`) to avoid corrupting SVG path data that contains sequences like `v6.24`.

### HTML Sanitization
Every save and load strips UI artifacts from section HTML. The selector list:
```
.section-edit-btn, .add-point, .table-controls, .svg-edit-hint, .block-delete
```
This prevents dead buttons, ghost add-points, and edit hints from being baked into Supabase.

### Exposing Functions Across Scripts
Scripts load in order. Later scripts expose functions on `window` for earlier scripts to call via null-check pattern:
```javascript
if (window._injectAddPoints) window._injectAddPoints(sec);
```
Exposed functions:
| Global | Defined in | Used by |
|---|---|---|
| `window._injectAddPoints` | Block Insertion | Section Editor |
| `window._removeAddPoints` | Block Insertion | Section Editor |
| `window._openAddPanel` | Block Insertion | Section Editor |
| `window._closeAddPanel` | Block Insertion | Section Editor |
| `window._updatePanelEditingState` | Block Insertion | Section Editor |
| `window._wireSectionFn` | Section Editor | TOC Manager |
| `window._triggerSectionSave` | Section Editor | Block Insertion (panel) |
| `window._triggerSectionCancel` | Section Editor | Block Insertion (panel) |
| `window._pushBlockUndo` | Block Insertion | Block Insertion |
| `window._blockUndoStack` | Block Insertion | Formatting Bar (Ctrl+Z) |
| `window.DossierApp` | DossierApp scaffold | All scripts |

### Supabase REST Pattern
Shared helper in DossierApp:
```javascript
DossierApp.sb('dossier_sections?id=eq.sec-01', {
  method: 'PATCH',
  body: JSON.stringify({ title: 'New Title', updated_at: new Date().toISOString() })
});
```
Storage uploads:
```javascript
DossierApp.storage('object/dossier-files/' + path, {
  method: 'POST',
  headers: { 'Content-Type': file.type },
  body: file
});
```

### CSS Design Tokens
```
--ink: #1a1a1a          (primary text)
--ink-mid: #555         (secondary text)
--ink-light: #888       (tertiary)
--ink-faint: #bbb       (disabled/hint)
--paper: #fafaf8        (background)
--signal: #c85a3a       (accent/action — burnt orange)
--signal-soft: rgba(200,90,58,0.06)
--done: #3a7d44         (success — green)
--amber: #c8963a        (warning)
--rule: #e0ddd8         (borders/dividers)
--mono: 'SF Mono', monospace
--sans: 'Inter', sans-serif
--serif: 'Playfair Display', serif
```

### Content Block Classes
| Class | Purpose | Grid/layout notes |
|---|---|---|
| `.prose` | Body text | `margin-left: 60px; max-width: 660px` |
| `.kv` | Key-value pairs | `margin-left: 60px;` DL grid |
| `.sub-head` | Numbered subsection | Grid: `60px 1fr` |
| `.sub-head.no-num` | Unnumbered heading | Grid: `1fr`, `margin-left: 60px` |
| `.figure` | Image/3D container | `margin-left: 60px` |
| `.stamp-card` | Note/callout box | `margin-left: 60px; border-left: 3px solid` |
| `.compare` | Comparison table | `margin-left: 60px; width: calc(100% - 60px)` |
| `.link-card` | URL preview card | `margin-left: 60px; max-width: 660px` |

---

## Issues Found & Fixed During Build

| Issue | Root cause | Fix |
|---|---|---|
| Long URLs overflow page | No `overflow-wrap` on content containers | Added `overflow-wrap: break-word` globally |
| TOC subtitles ≠ section tags | Manual text drift | Reconciled all 9 sections, added Supabase source-of-truth reconciliation on load |
| Ghost edit buttons after reload | `sec.innerHTML` saved with buttons inside | Sanitize on save (strip buttons) + sanitize on load + `wireSection` strips before adding |
| Edit button rendering glitch | 7px font + 1px border = checkbox appearance | Bumped to 9px, kept bordered green box style per preference |
| Ctrl+Z broken globally | § 08 tracker's global `Ctrl+Z` handler called `preventDefault` on all sections | Scoped to only fire when `sec-08.contains(document.activeElement)` |
| Ctrl+Z broken in contenteditable | Explicit `execCommand('undo')` with `preventDefault` intercepted browser native undo | Removed explicit handler entirely, browser handles natively |
| SVG editable without double-click | CSS `pointer-events: none` doesn't override `contentEditable` | Set `contenteditable="false"` on `.figure`/`.fig-frame` elements via JS |
| Add-points invisible | `height: 0; opacity: 0` with no hover zone | Changed to `padding: 10px 0; opacity: 0.5` in edit mode |
| Section outline looked solid | `1px dashed` too thin on retina | Changed to `2px dashed` with `12px` offset |
| Heading block text cramped | `.sub-head` grid expects 2 columns, heading only had 1 | Added `.sub-head.no-num` variant: `grid-template-columns: 1fr` |
| Table controls invisible | CSS `+` sibling selector broken by add-point injection between table and controls | Changed to `section.editing .table-controls { opacity: 1 }` |
| ADD panel blocks content | Grey overlay intercepted all clicks | Removed overlay, panel is persistent non-blocking sidebar |
| Save button lost on scroll | Sticky header only works within section bounds | Added Save/Cancel to the ADD panel (fixed-position, always visible) |
| SVG path corrupted by version bump | `sed "s/v6.2/v6.3/g"` matched `v6.24` in SVG path data | Use context-aware sed patterns targeting surrounding text |
| No way to edit uploaded figure caption | Only delete was available, no edit flow | Added ✎ button → inline title/desc editing with Save |
| Confirm modals feel hostile | `window.confirm()` is jarring | Two-state click: first click arms (red "delete?"), second click executes, auto-resets after 2s |

---

## Known Limitations / Future Work

- **Post-save undo** — once you click Save, changes are committed to Supabase. No rollback mechanism. Could add a version history table.
- **Block drag-to-reorder** — blocks can be inserted and deleted but not dragged to new positions. Would need drag handles + DOM reordering.
- **SketchUp files** — `.skp` uploads to storage but renders as placeholder (no browser-native viewer). Only GLB gets the Three.js treatment.
- **Link preview OG fetching** — link cards are manually filled. Auto-fetching OG meta from URLs would need a Supabase Edge Function or proxy.
- **Collaborative editing** — both users see saved changes on reload, but there's no real-time sync or conflict resolution.
- **Old Supabase data** — sections saved before v7 may contain stale placeholder text ("upload coming in Phase 5") or old section-tag text. The sanitization + reconciliation handles buttons and tags, but content-level placeholders require manual cleanup.
