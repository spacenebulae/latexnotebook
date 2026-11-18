<div align="center">

# ✧・゚: *✧ LaTeX Notebook ✧*:・゚✧

### one (1) html file that quietly became an entire note-taking app (｡•̀ᴗ-)✧

<img src="https://img.shields.io/badge/shape-single%20html%20file-ff69b4" alt="single file">
<img src="https://img.shields.io/badge/build%20step-none%2C%20never-88e" alt="no build">
<img src="https://img.shields.io/badge/renderer-KaTeX%200.19.0-00338e" alt="KaTeX">
<img src="https://img.shields.io/badge/sync-Supabase%20v2-3ecf8e" alt="Supabase">
<img src="https://img.shields.io/badge/autocorrect%20for%20math-yes%20really-ffb6c1" alt="autocomplete">

*a split-pane text + LaTeX editor with accounts, cloud sync, multiple notebooks, tab-autocomplete for ~175 math commands, an infinite-ish zoomable preview, and one-click PNG / PDF export. all in one file. because why not (˘ω˘)*

> it doesn't ask for much. it doesn't install anything. it just opens in a browser and starts existing.

</div>

---

## 📖 table of contents

- [what is this](#-what-is-this)
- [quick start](#-quick-start)
- [feature tour](#-feature-tour)
- [math syntax cheat sheet](#-math-syntax-cheat-sheet)
- [keyboard & mouse map](#️-keyboard--mouse-map)
- [how it thinks (architecture)](#-how-it-thinks-architecture)
- [function reference](#-function-reference-the-big-part)
  - [1 · configuration & constants](#1--configuration--constants)
  - [2 · DOM elements](#2--dom-elements)
  - [3 · storage helpers](#3--storage-helpers)
  - [4 · theme & font](#4--theme--font)
  - [4B · preview zoom & pan](#4b--preview-zoom--pan)
  - [5 · the incremental renderer](#5--the-incremental-renderer)
  - [6 · math autocomplete](#6--math-autocomplete)
  - [7 · notebooks & sync](#7--notebooks--sync-the-serious-part)
  - [8 · authentication](#8--authentication)
  - [9 · export machinery + PNG](#9--export-machinery--png)
  - [10 · PDF export](#10--pdf-export)
  - [11 · initialization](#11--initialization)
- [localStorage key map](#-localstorage-key-map)
- [things the status fairy says](#-things-the-status-fairy-says)
- [tech stack](#-tech-stack)
- [known quirks](#-known-quirks)
- [license](#-license)

---

## ✨ what is this

you type plain text on the left. if you wrap something in `$...$`, it becomes math on the right. that's the core loop, and it renders **live while you type** — no button, no waiting, no ceremony.

but then it kept going:

| it has | because |
| --- | --- |
| 📓 multiple notebooks per account | one giant document is chaos |
| ☁️ Supabase accounts + cloud sync | switching devices shouldn't hurt |
| 🌙 dark mode (remembers you) | obviously |
| ✍️ 7 selectable editor fonts | your editor, your aesthetic |
| 🔮 tab-autocomplete inside math mode | `\alpha` is 6 keystrokes too many |
| 🔍 zoom (20%–800%) + pan + custom scrollbars | reading huge matrices shouldn't require squinting |
| 🖼️ one-click PNG export at 3× resolution | for slides, chats, whatever |
| 📄 configurable multi-page PDF export | because screenshots of PDFs are sad |
| 🛟 offline mode, retries, conflict resolution | the cloud is not always available, and that's fine |

no build step. no `npm install`. no webpack config that only works on full moons. open the file, sign in, write math. that's the whole ritual ✧

---

## 🚀 quick start

1. **put `index.html` somewhere a browser can reach it.** GitHub Pages, any static host, or literally double-click it from your desktop. it's not picky (¬‿¬)

2. **make a [Supabase](https://supabase.com) project.** the app talks to it through the browser — there is no backend of ours.

3. **create a `documents` table.** the app expects these columns (this is the shape it reads/writes; tweak types to taste):

   ```sql
   create table documents (
     user_id     uuid primary key references auth.users,
     content     text not null default '',
     updated_at  timestamptz not null default now()
   );
   ```

4. **turn on Row Level Security** so people can only touch their own notebook:

   ```sql
   alter table documents enable row level security;

   create policy "own rows only" on documents
     for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
   ```

5. **paste your project URL + publishable key** into the two constants at the top of the `<script>` (`SUPABASE_URL`, `SUPABASE_KEY`).

6. open the page → **Create Account** → confirm the email → sign in → write math.

> no Supabase configured? the page will still load, it'll just sit at the sign-in screen looking sad. the app requires an account before the editor appears — everything downstream is per-user.

---

## 🎀 feature tour

### the editor
a plain `<textarea>` with a very overengineered conscience. every keystroke: updates the active notebook, re-renders *only the changed blocks* of the preview (diff-based, not full re-render), saves locally after **80 ms** of calm, and pushes to the cloud after **600 ms**. typing fast doesn't melt anything.

### the preview
- **live render** of `$...$`, `$$...$$`, `\(...\)`, `\[...\]`, and bare `\begin{pmatrix}...\end{pmatrix}`-style environments.
- **Render: On/Off toggle** — pause rendering while typing a huge doc, then hit **Render once** when you're ready.
- **zoom** with plain scroll anchored at your cursor, `+` / `−` / `100%` HUD buttons, **spacebar-held** or **middle-mouse drag** to pan, plus a hand-tool toggle. range 20%–800%.
- **handmade scrollbars** (vertical + horizontal) drawn as overlays with draggable thumbs, because a scaled-and-translated surface confuses native scrollbars.

### autocomplete (the cute part)
type `\` inside math mode and a little popup appears with glyphs, descriptions and a **Tab** badge. arrow keys walk the list, Tab commits, Esc dismisses (it remembers per-token so it doesn't nag). a ghost-gray inline suggestion shows the rest of the top match directly in the text. used commands **rank higher over time** — it learns what you like. uwu

### notebooks
a hamburger menu in the toolbar lists all your notebooks. **click** to switch, **right-click a name** to rename inline, **×** to delete (asks first if there's content), **+ New** to make another. each notebook is a separate document, all synced as one store.

### exports
- **PNG** — the entire rendered preview as one high-res image (3×, auto-clamped to browser canvas limits).
- **PDF** — a modal with page size, orientation, margins, theme, font size, quality, optional header title and page numbers. pagination is *smart*: it measures real line boxes and KaTeX node boxes so it doesn't slice an equation in half.

### sync & survival
local snapshot always lands first (your text is never lost waiting for a network), cloud save follows with retries (2s → 5s → 10s → 20s → 30s backoff), a `keepalive` fetch fires on tab close, other tabs sync through the `storage` event, and if the cloud has something newer than your unsaved local edits you get a **conflict banner** with three gentle choices. see [notebooks & sync](#7--notebooks--sync-the-serious-part) for the full state machine.

---

## 🧮 math syntax cheat sheet

| you write | you get |
| --- | --- |
| `Euler wrote $e^{i\pi} + 1 = 0$ once` | inline math inside a sentence |
| `$$ \int_0^1 x^2 dx = \frac{1}{3} $$` | centered display math |
| `\( ... \)`  /  `\[ ... \]` | inline / display math, LaTeX-purist flavor |
| `\begin{pmatrix} a & b \\ c & d \end{pmatrix}` | a whole *standalone block* — no `$$` wrapper needed |
| `\$5.00` | a literal dollar sign (escaped `$` is never treated as math) |

**standalone-capable environments** (rendered as display math even outside `$$`):
`matrix`, `matrix*`, `pmatrix`, `pmatrix*`, `bmatrix`, `bmatrix*`, `Bmatrix`, `Bmatrix*`, `vmatrix`, `vmatrix*`, `Vmatrix`, `Vmatrix*`, `smallmatrix`, `cases`, `dcases`, `rcases`, `drcases`, `aligned`, `alignedat`, `gathered`, `align`, `align*`, `equation`, `equation*`, `gather`, `gather*`, `multline`, `multline*`, `array`, `darray`, `split`, `CD`.

> escaped delimiters, nested braces, `\left(`-style pairs — the parser tracks all of it so a `\{` inside math doesn't end your `\{`-less `}` … you get it. it's careful (•̀ᴗ•́)و

---

## ⌨️ keyboard & mouse map

| input | action |
| --- | --- |
| `Tab` | accept the top autocomplete suggestion (math mode only) |
| `↑` / `↓` | cycle suggestions in the popup |
| `Esc` | dismiss suggestion for this token · close PDF modal · close notebooks menu |
| `Ctrl/Cmd + S` | flush unsaved changes to cloud, then export the PNG (yes, really) |
| `scroll` over preview | exponential zoom anchored at the cursor (20%–800%) |
| `Space` (held, focus not in a field) | switch to pan-mode cursor — drag to pan |
| `middle-mouse drag` | pan |
| `+` / `−` / `100%` buttons | zoom in/out/reset · reset also clears pan |
| ✋ button | toggles the persistent pan hand-tool |
| `right-click` a notebook name | rename it inline (`Enter` saves, `Esc` cancels) |
| `Enter` / `Esc` while renaming | commit / abort the rename |

---

## 🧠 how it thinks

```
your keystroke
   │
   ├─► onEditorContentChanged()
   │      ├─ writes text into the active notebook object
   │      ├─ schedulePreviewRender()  ──► diff render (only changed blocks)
   │      ├─ scheduleSaves() ──► local (80 ms debounce) ──► cloud (600 ms debounce)
   │      └─ updateAutocomplete()  ──► maybe a ghost + popup appears
   │
   └─ on tab close / hide / unload
          ├─ flushLocalDraftSync()   (localStorage snapshot, always)
          └─ sendKeepaliveCloudFlush() (fetch keepalive POST, if < 60 KB & dirty)
```

**rendering is incremental.** the document is split into *blocks* (paragraph-level chunks that never straddle an open math region). new blocks are diffed against the rendered ones by longest common prefix/suffix, and only the changed middle is re-rendered. big insertions (> 16 blocks) are time-sliced across animation frames with an 8 ms budget so the UI never stutters. rendered KaTeX HTML is LRU-cached (800 entries) so re-typing the same expression is free.

**sync is layered.** localStorage is the source of truth for "right now"; the cloud is the source of truth for "everywhere else". meta in localStorage remembers `dirty` + the last synced content, which is what makes conflict detection and offline recovery possible without a server telling us anything.

---

# 📚 function reference (the big part)

> the file is one big IIFE, organized into 11 numbered sections. everything below lives inside it. here's the whole cast, in order (´｡• ω •｡`)

## 1 · configuration & constants

not functions, but the knobs everything else reads:

| name | value | meaning |
| --- | --- | --- |
| `SUPABASE_URL` / `SUPABASE_KEY` | — | your project URL + publishable key |
| `LOCAL_SAVE_DEBOUNCE_MS` | 80 | wait after typing before writing localStorage |
| `CLOUD_SAVE_DEBOUNCE_MS` | 600 | wait after typing before pushing to the cloud |
| `RETRY_DELAYS_MS` | `[2s, 5s, 10s, 20s, 30s]` | backoff ladder for failed cloud saves |
| `FOCUS_SYNC_MIN_INTERVAL_MS` | 10000 | min gap between focus-triggered cloud checks |
| `RENDER_LARGE_DOC_THRESHOLD` | 8000 | chars above which render debounce relaxes (16 → 75 ms) |
| `RENDER_FAST_DEBOUNCE_MS` / `RENDER_LARGE_DEBOUNCE_MS` | 16 / 75 | typing debounce before a diff-render |
| `RENDER_FRAME_BUDGET_MS` | 8 | max main-thread time per time-sliced render frame |
| `KATEX_CACHE_LIMIT` | 800 | LRU entries of cached rendered math |
| `EXPORT_SCALE` | 3 | default PNG export resolution multiplier |
| `MAX_CANVAS_DIMENSION` / `MAX_CANVAS_PIXELS` | 30000 / 90,000,000 | browser canvas ceiling guards |
| `STORAGE_RENDER_KEY` | `latex_editor_render_enabled` | localStorage key for the render toggle |
| `STORAGE_PDF_OPTS_KEY` | `latex_editor_pdf_options` | localStorage key for saved PDF preferences |

**also, a pre-paint bootstrap IIFE in `<head>`**: defines the `FONTS` map (7 fonts), a `safeGet()` wrapper, `window.applyEditorFont(key)` (sets the `--app-font` / `--app-font-weight` CSS variables), and applies the saved theme + font *before first paint* so there's no flash of wrong theme. `window.LATEX_EDITOR_FONTS` is its parting gift to the main script.

---

## 2 · DOM elements

one flat block of `document.getElementById(...)` — about 60 refs covering the editor, preview, toolbar, notebooks menu, zoom HUD, scrollbars, ghost, autocomplete popup, every PDF modal field, and the auth panel. no framework, no query-selector archaeology, just names. it's very honest code (￣▽￣)ゞ

---

## 3 · storage helpers

| function | what it does |
| --- | --- |
| `safeStorageGet(key)` | `localStorage.getItem` wrapped in try/catch — returns `null` instead of throwing in paranoid browser modes |
| `safeStorageSet(key, value)` | the write-side twin; logs and returns `false` on failure |
| `getLocalDocumentKey(userId)` | → `latex_editor_content_<userId>` |
| `getLocalMetaKey(userId)` | → `latex_editor_meta_<userId>` |
| `readLocalSnapshot(userId)` | reads `{ content, meta }` for a user; tolerates corrupted meta JSON |
| `writeLocalSnapshot(userId, content, metaUpdates)` | writes content and *merges* meta updates over the existing meta (never clobbers sibling fields) |

---

## 4 · theme & font

| function | what it does |
| --- | --- |
| `initFontPicker()` | fills the font `<select>` from `window.LATEX_EDITOR_FONTS`, restores the saved font, applies it, and persists on change (also refreshes autocomplete geometry — fonts change text metrics) |
| `currentTheme()` | reads `data-theme` off `<html>` → `'dark'` or `'light'` |
| `updateThemeButton()` | syncs the moon/sun button tooltip + `aria-pressed` state |
| `initThemeToggle()` | wires the click: flip theme, persist to `latex_editor_theme`, update the button |

fonts on offer: **Nunito ExtraBold** (default), Verdana, Helvetica, JetBrains Mono, Calibri, Arial, Consolas. the *editor* font changes; the *rendered preview* keeps a neutral system font so math always looks like math.

---

## 4B · preview zoom & pan

module state: `previewZoom`, `previewPanX/Y`, `isPanToolActive`, `isSpacePanHeld`, `panDragState`, bounds `MIN_PREVIEW_ZOOM = 0.2` … `MAX_PREVIEW_ZOOM = 8.0`.

| function | what it does |
| --- | --- |
| `getPreviewPanBounds()` | computes viewport vs scaled-content bounds and the pan limits (keeps ≥ 80px / 50% of content visible on each axis so you can't lose the document in the void) |
| `clampPreviewPan()` | squeezes pan into those limits; at 100% zoom snaps back to normal scroll-like behavior |
| `getScrollbarMetricsY(b)` / `getScrollbarMetricsX(b)` | thumb size + scroll range math for the vertical / horizontal overlay scrollbar |
| `updatePreviewScrollbars()` | shows/hides each custom scrollbar and positions its thumb from current pan |
| `applyPreviewTransform()` | clamps pan, applies `translate() scale()` to the preview surface, updates the `100%` button label and scrollbars — called after literally everything |
| `zoomPreviewAtViewportPoint(nextZoom, mx, my)` | clamps the zoom, then adjusts pan so the point under the mouse *stays* under the mouse (anchored zoom) |
| `resetPreviewZoom()` | back to 1.0 / no pan |
| `syncPanCursorState()` | toggles the `grab` cursor + hand-tool button `aria-pressed` when panning is available |
| `initPreviewZoomAndPan()` | wires it all: wheel-zoom (handles `deltaMode` line/page units), middle/left+tool/space drag-pan, scrollbar thumb dragging, `+`/`−`/reset buttons, hand-tool toggle, spacebar hold (ignored while typing in fields), and a `ResizeObserver` to re-clamp on resize |

---

## 5 · the incremental renderer

the pride of the file (¬‿¬). state: `isRenderEnabled`, `renderedBlocks[]`, debounce/RAF handles, a `renderGeneration` counter (to cancel stale async work), and the `katexCache` LRU Map.

| function | what it does |
| --- | --- |
| `getCachedKatexHtml(expr, displayMode)` | LRU-cached `katex.renderToString` (touch-on-read, evict-oldest past 800 entries); renders with `throwOnError: false`, `strict: false` so bad input degrades instead of exploding |
| `findEndOfMath(delimiter, text, startIndex)` | scans for the closing `$`/`$$`/`\]`/`\)` while respecting brace nesting and `\\` escapes (so `\}` inside math doesn't confuse it) |
| `isEscapedAt(text, pos)` | is the char at `pos` preceded by an odd number of backslashes → escaped |
| `findTopLevelEnvEnd(text, envName, searchStart)` | finds the matching `\end{env}` for a `\begin{env}` with correct nesting depth |
| `appendKatexToFragment(frag, expr, displayMode, rawFallback)` | renders (cached) → template → fragment; on any error, falls back to the raw LaTeX as plain text so a broken formula never eats the document |
| `buildRenderedFragment(blockText)` | the mini-parser: walks a block, honoring `$…$`, `$$…$$`, `\(…\)`, `\[…\]`, escaped `\$`, and standalone `\begin{...}` environments; plain text runs become text nodes |
| `createRenderedBlockElement(blockText)` | wraps a rendered fragment in a `<span class="preview-block">` — one block = one DOM unit |
| `splitDocumentIntoBlocks(text)` | line-scanner that groups the doc into blocks which never straddle an open math region (tracks inline/display/bracket/paren modes + a standalone-env stack + nesting depth); includes safeguards: a stray inline `$` auto-closes after blank lines / 6 lines, and *everything* force-closes after 100 blank lines so an unclosed delimiter can't swallow the whole document |
| `cancelPendingRenderWork()` | bumps the generation counter, kills the debounce timer and pending RAF — used whenever newer input arrives |
| `applyIncrementalPreviewRender(text, immediate)` | the diff renderer: split → longest common prefix/suffix vs `renderedBlocks` → splice out removed, insert the changed middle. ≤ 16 blocks (or `immediate`) inserts synchronously; more than that is time-sliced over `requestAnimationFrame` in ≤ 12-block / 8 ms batches. debounce is 16 ms normally, 75 ms for docs > 8000 chars |
| `clearRenderedPreview()` | cancels work, empties the preview, resets transform state |
| `syncRenderToggleUI()` | updates the Render: On/Off button + `aria-pressed` + paused-banner visibility |
| `schedulePreviewRender({force, immediate})` | the throttle entry point — no-ops while rendering is paused (unless forced) |
| `initRenderToggle()` | the On/Off button (persisted) and the paused-banner's **Render once** button |

---

## 6 · math autocomplete

state: the `MATH_COMMANDS` catalog (~175 entries: `cmd`, full `insert` text, `cursorBack` offset, a unicode `glyph`, one-line `desc`, popularity `rank`), `MATH_ENVIRONMENTS` (for mode detection), a `commandUsageBoost` Map (learning!), and `acState` (active, token, matches, selected index, dismissed-token signature).

| function | what it does |
| --- | --- |
| `isInMathMode(text, cursorPos)` | replays the text up to the caret tracking `$`/`$$`/`\(`/`\[`/env depth (with a blank-line auto-close rule) → is the caret inside math? this is what stops autocomplete from triggering while you're writing plain prose |
| `findCommandTokenBeforeCaret(text, caretPos)` | regex-extracts the half-typed `\command` (or `\begin{…`, `\left(`) immediately before the caret; rejects escaped tokens |
| `rankAndMatchCommands(token)` | scores every catalog entry: prefix match on insert (`+60`), exact match (`+25`), popularity rank, usage boost, minus a small penalty for long inserts; returns top 7. bare `\` returns the top 7 overall — your personal greatest hits |
| `measureCaretCoordinates(caretPos)` | the fun one: copies the textarea's font metrics onto a hidden `#caretMirror` div, injects the text-before-caret + a zero-width marker + a peek at the text after, then reads the marker's `offsetLeft/offsetTop` → real caret x/y, even with wrapping and scrolling |
| `hideAutocomplete()` | clears state, hides ghost + popup |
| `renderAutocompleteUI()` | positions the ghost inline suggestion (only if the caret line is visible and the next char is clear) and builds the popup rows (glyph / command with dim completion / description / Tab badge); flips the popup above the caret if it would hit the bottom of the window; clamps to the viewport edges; scrolls the active row into view |
| `updateAutocomplete()` | the gatekeeper: signed-in? caret collapsed? in math mode? token found? not dismissed for this token? → then rank, try to preserve the previously selected command across list refreshes, and paint. hides otherwise |
| `acceptAutocomplete(indexOverride?)` | replaces the token with the catalog entry's insert text (via `execCommand insertText` for native undo support, falling back to `setRangeText`), places the caret at `cursorBack`, bumps that command's usage boost (+5, cap +30), hides the UI, and notifies the change pipeline |

**the catalog, by vibes:** ~30 structures & fractions (`\frac`, `\sqrt`, `\overbrace`, `\begin{aligned}`…) · ~15 big operators & calculus (`\sum`, `\int_{}^{}`, `\oint`…) · 27 lowercase greek · 11 uppercase greek · ~40 relations, sets & logic (`\leq`, `\in`, `\implies`…) · ~19 binary operators & arrows · ~36 spacing, ellipsis & functions (`\dots`, `\sin`, `\hbar`, `\pmod`…).

---

## 7 · notebooks & sync *(the serious part)*

state: `currentUser`, `activeUserId`, `latestAccessToken`, a `documentLoadId` guard, readiness flags (`isDocumentReady`, `hasUnsavedChanges`, `hasUnresolvedConflict`, `needsCloudReconcile`), `docRevision`, `lastSyncedContent/At`, the debounce/retry timers, and the `notebookStore` itself.

the notebook store on disk is a versioned JSON (`__latexNotebookFormat: 2`) holding all notebooks + which is active; a legacy plain-text document is auto-migrated into a one-notebook store named *Main Notebook* on first load.

### notebook management

| function | what it does |
| --- | --- |
| `generateNotebookId()` | `nb_<base36 time>_<random>` ids |
| `getNextNotebookName()` | first unused "Notebook N" |
| `getActiveNotebook()` | resolves `activeId` → notebook object (falls back to the first) |
| `parseNotebookStore(rawContent)` | parses/validates the store JSON; repairs missing ids/names; falls back to a fresh single-notebook store for legacy or garbage input |
| `serializeNotebookStore()` | stamps the current editor text into the active notebook → JSON string |
| `mergeNotebookStores(localRaw, cloudRaw)` | union by id; an empty side's content is replaced by the non-empty side; two real different contents → local + `\n\n--- Cloud Version ---\n\n` + cloud; duplicate names get `" (Cloud)"` suffixes |
| `updateActiveNotebookLabel()` | toolbar label + tooltip |
| `openNotebooksMenu()` / `closeNotebooksMenu()` | dropdown visibility + `aria-expanded` |
| `renderNotebooksList()` | rebuilds the notebook rows (active dot, name, hover × delete, click-to-switch, right-click-to-rename) |
| `startRenamingNotebook(nbId, rowEl)` | swaps the name span for an inline input — `Enter` commits, `Esc` cancels, blur commits; stops click/contextmenu from bubbling |
| `switchNotebook(nbId)` | saves current text → sets active → loads target text → immediate re-render → zoom reset → save scheduled |
| `createNotebook()` | stamps current content, appends an empty notebook, activates it, scrolls the list to it |
| `deleteNotebook(nbId)` | refuses when it's the last one; `confirm()` if it has content; if you deleted the active one, activates a neighbor |

### sync engine

| function | what it does |
| --- | --- |
| `setStatus(text, title?)` | the status fairy speaks (bottom-right badge) |
| `clearAllSaveTimers()` | kills local / cloud / retry timers |
| `hideSyncBanner()` | hides the yellow warning banner |
| `showCloudRetryBanner(message)` | banner with a **Retry Cloud Load** button for unreachable-on-load |
| `showConflictBanner(cloud, local, cloudUpdatedAt)` | the three-button conflict dialog (below) |
| `isAuthOrJwtError(error)` | 401 / `PGRST301` / jwt-expired sniffing |
| `selectCloudDocumentWithAuthRetry(userId)` | `select … maybeSingle()` on `documents`; on auth error, refreshes the session once and retries |
| `upsertCloudDocumentWithAuthRetry(userId, content, updatedAt)` | the upsert twin (conflict target `user_id`), same refresh-retry logic |
| `flushLocalDraftSync()` | writes the serialized store + dirty meta to localStorage immediately |
| `scheduleCloudRetry()` | schedules the next backoff retry; routes to *reconcile* or *resave* depending on what's pending |
| `flushCloudSave()` | the main save loop: while dirty → serialize → local snapshot → upsert → bookkeeping (synced content/at, reset retry ladder). queues re-runs if called mid-save (`resaveQueued`), guards against user switches and revision races, and reports offline-vs-error status honestly |
| `reconcileCloudAfterOffline()` | after being offline: fetch cloud; identical or clean-local → adopt cloud; local dirty on top of the same cloud base → just push; anything else → conflict banner |
| `scheduleSaves()` | the 80 ms local + 600 ms cloud debounce pair; bumps `docRevision`, marks dirty, clears pending retries |
| `sendKeepaliveCloudFlush()` | tab-is-closing insurance: raw `fetch(…, {keepalive: true})` POST of the document (< 60 KB only); silently best-effort |
| `checkCloudUpdatesOnFocus()` | at most every 10 s, when idle + clean: pull the cloud doc; if it differs, adopt it and re-render. silent on failure — no nagging |
| `loadUserDocument(user)` | the login-time conductor: reads local snapshot, fetches cloud, then decides — no cloud doc → migrate local up; local dirty on the same base → push; local dirty on a *different* cloud → conflict banner; otherwise adopt cloud. on network failure, serves the local copy and schedules reconcile. every async step re-checks `documentLoadId` so a logout-login race can't apply stale results |
| `onEditorContentChanged()` | the keystroke hub: notebook content + timestamp → schedule render → schedule saves → autocomplete update |

**the conflict banner's three futures:**

| button | what happens |
| --- | --- |
| **Keep Local & Sync** | your local text wins, gets pushed to the cloud |
| **Use Cloud Version** | cloud wins; your local version is backed up to `latex_editor_conflict_backup_<userId>` first (nothing is ever thrown away) |
| **Keep Both** | `mergeNotebookStores` concatenates both versions into the same notebook (with a `--- Cloud Version ---` separator) and pushes |

---

## 8 · authentication

| function | what it does |
| --- | --- |
| `setAuthButtonsDisabled(disabled)` | disables sign in / create account during a request |
| `getEmailRedirectUrl()` | returns the current origin + path as the email-confirmation redirect (only on http/https, not `file://`) |
| `initAuth()` | wires everything: form submit → `signInWithPassword`; **Create Account** → `signUp` (+ redirect URL, with a "check your email" message when confirmation is required); **Sign Out** → flushes local, saves to cloud first if dirty, then `signOut({scope:'local'})`. the `onAuthStateChange` callback is the real heart: on login it resets per-user state and calls `loadUserDocument`; on logout it tears down everything (banners, timers, editor, notebooks UI) and shows the sign-in panel. a 250 ms reveal-timer shows the panel if auth never resolves, and a missing Supabase SDK yields a clear error message instead of a silent trap |

---

## 9 · export machinery + PNG

lazy-loaded libraries (html2canvas 1.4.1, jsPDF 3.0.4), a cached inlined-KaTeX-CSS pipeline, and a clone-and-measure approach: exports render a *fresh hidden clone* of the preview, so whatever is (or isn't) currently on screen doesn't matter.

| function | what it does |
| --- | --- |
| `loadHtml2Canvas()` / `loadJsPdf()` | promise-cached script injection from CDN; errors clear the cache so a retry can try again |
| `chooseExportScale(width, height, preferredScale)` | clamps the export scale against canvas limits (30,000 px per side / 90 M pixels total) |
| `arrayBufferToBase64(buffer)` | chunked (`0x8000`) byte-to-binary + `btoa` — safe for big font files |
| `getInlinedKatexCss()` | fetches `katex.min.css`, finds every `woff2` `url(...)`, fetches each font, and rewrites the CSS with base64 data-URIs — so exported clones keep math fonts without external requests. cached in a promise |
| `createExportSurfaceClone(options)` | builds the offscreen export div: fixed width, optional title heading, freshly rendered blocks (re-split + re-render from the live text, not the DOM), background/color/font-size/padding overrides, invisible and `pointer-events: none` |
| `measureExportDimensions(clone)` | union of the clone rect + scroll extents + every KaTeX/svg node rect (some KaTeX internals overflow their boxes) → full width/height + any content poking above the top |
| `ensureTopPaddingForOverflow(clone)` | if math overflows the top, adds matching padding and re-measures (one rAF each round) so nothing is clipped |
| `prepareCloneForHtml2Canvas(root)` | html2canvas-specific first aid: removes `.katex-mathml` (duplicate invisible math), forces sub-1px fraction lines to 1px, pins explicit fills/strokes on SVGs, fixes `.hide-tail` (sqrt tail) overflow clipping, bakes explicit svg width/height |
| `renderCloneViaForeignObject(clone, width, sliceHeight, fullHeight, scale, bgColor, sliceY)` | the primary rasterizer (non-Safari): serializes the clone into an SVG `<foreignObject>` with the inlined CSS inside `<style>`, loads it as an image, and blits it to a canvas at the target scale. `sliceY` enables rendering just a horizontal slice — that's how multi-page PDF works |
| `renderCloneWithHtml2Canvas(clone, …)` | the fallback rasterizer: direct html2canvas for slice 0, or a hidden viewport div with the clone offset by `-sliceY` for later slices |
| `renderCloneToCanvas(clone, …)` | tries foreignObject first, falls back to html2canvas (Safari is detected and skips straight to the fallback — its foreignObject support is a cursed house) |
| `canvasToBlob(canvas)` | `toBlob('image/png')` promise-wrapped |
| `exportPreviewAsPng()` | the whole PNG dance: wait for `document.fonts.ready` → clone → measure → scale → rasterize → blob → download as `<notebook name>.png` (sanitized) → status reports final dimensions, or the error, or "scaled down to fit browser limits" if the clamp kicked in |

---

## 10 · PDF export

page sizes in points (`letter 612×792`, `a4 595.28×841.89`, `legal 612×1008`, `a5 419.53×595.28`), margins 10/20/30/0 mm, quality 2×/3×, font sizes 12/15/18 px, theme light-or-current.

| function | what it does |
| --- | --- |
| `mmToPt(mm)` | millimeters → PDF points (`× 2.83464567`) |
| `parseRgbColor(cssColor, fallback)` | regex-parses `rgb()` / `rgba()` strings |
| `getPdfOptionsFromForm()` | reads + sanitizes every modal field (filename chars stripped, safe defaults for anything malformed) → options object |
| `savePdfOptionPreferences()` / `loadPdfOptionPreferences()` | persist / restore PDF settings in localStorage (not the filename/title — those are per-export) |
| `updatePdfSummary()` | the little `Letter · Portrait · 20 mm margins · …` preview line under the form |
| `openPdfModal()` | remembers the focused element for restoration, pre-fills the filename from the notebook name (unless you edited it), shows the modal, and *pre-warms* the jsPDF + inlined CSS loads in the background so the actual export is fast |
| `closePdfModal()` | hides it, restores the previously focused element |
| `collectIndivisibleLineBoxes(container)` | measures every line of text (via TreeWalker + Range rects) and every KaTeX node union-box → the "don't cut here" list |
| `computeSmartPageSlices(totalHeight, pageContentHeightPx, boxes)` | walks page-height windows; wherever a box would be sliced in half, moves the break up to that box's top (never advancing less than 20% of a page, 50-iteration guard) → page slices that respect math |
| `exportPreviewAsPdf(event)` | the maestro: compute printable area → clone at a CSS px width matching the PDF width → measure → smart slices → new jsPDF → then either **one big canvas** sliced per page (if it fits the canvas limits) or **per-page sliced rendering** (huge docs) → each page: fill background, draw the slice, optional `Page i of n` footer in a contrast-aware gray → `doc.save(name + '.pdf')`. restores buttons and closes the modal in `finally` |
| `initPdfModal()` | wires open/close (backdrop click, Esc, cancel — all disabled while an export runs), change listeners for the summary line, marks the filename as user-edited on input, and submits the form to `exportPreviewAsPdf` |

---

## 11 · initialization

the last stretch — the IIFE calls home:

`initFontPicker()` → `initThemeToggle()` → `initPreviewZoomAndPan()` → `initRenderToggle()` → `initPdfModal()` → `initAuth()`

then the global wiring, in order:

| listener | behavior |
| --- | --- |
| notebooks hamburger / `+ New` / outside-click | open/close dropdown, create notebook |
| editor `input` | `onEditorContentChanged` (the whole pipeline above) |
| editor `keydown` | Tab / ↑ / ↓ / Esc for the autocomplete |
| editor `keyup` (arrows, Home/End) + `click` | re-evaluate autocomplete for the new caret |
| editor `scroll` / window `resize` | re-position the popup (or hide it if the caret row scrolled out of view) |
| editor `blur` | hides autocomplete after a 120 ms grace (so clicking a suggestion still works) |
| `DOMContentLoaded` | force an immediate render if already signed in and rendering is on |
| `visibilitychange` (hidden) | local flush + cloud flush if dirty |
| `visibilitychange` (visible) / window `focus` | `checkCloudUpdatesOnFocus` |
| `pagehide` | local flush + keepalive cloud flush |
| `beforeunload` | local flush + keepalive flush, and shows the native "wait!" prompt if a cloud save is mid-flight or dirty while online |
| `online` / `offline` | reconcile or resave on reconnect; honest offline status |
| `storage` (cross-tab) | another tab saved → this tab adopts the doc (only when this tab is clean) |
| global `keydown` | Esc closes modal/menu; `Ctrl/Cmd+S` flush-then-export-PNG |

---

## 🔑 localStorage key map

| key | holds |
| --- | --- |
| `latex_editor_theme` | `'light'` / `'dark'` |
| `latex_editor_font` | one of the 7 font keys |
| `latex_editor_render_enabled` | `'true'` / `'false'` |
| `latex_editor_pdf_options` | JSON of saved PDF modal settings |
| `latex_editor_content_<userId>` | the serialized notebook store (all notebooks) |
| `latex_editor_meta_<userId>` | JSON: `dirty`, `updatedAt`, `syncedContent`, `syncedAt` |
| `latex_editor_conflict_backup_<userId>` | your local version, saved when you pick "Use Cloud Version" in a conflict |

---

## 🎀 things the status fairy says

| message | when |
| --- | --- |
| `Typing…` | you're, well, typing |
| `Saving to cloud…` / `Saved to cloud` | the 600 ms debounce fired / round-trip finished |
| `Offline — saved locally, will sync when online` | offline with unsaved changes |
| `Cloud save retrying, local copy saved: …` | a save failed; local copy is safe |
| `Sync conflict needs review` | the banner wants your attention |
| `Loaded from cloud` / `Synced from cloud` / `Updated from cloud` | login load / offline reconcile / focus-check adoption |
| `Synced from another tab` | the `storage` event brought a newer doc |
| `Local document migrated to cloud` | first cloud save of an existing local doc |
| `Cloud unreachable — using local copy (will sync when online)` | login failed but local exists |
| `Exported full preview: W × H px` | PNG saved (mentions any scale-down) |
| `Exported PDF (n pages)` | PDF saved |
| `Signed out` | it's just us now |

---

## 🧰 tech stack

| thing | version | for |
| --- | --- | --- |
| [KaTeX](https://katex.org) | 0.19.0 | math rendering (SRI-pinned CDN) |
| [Supabase JS](https://github.com/supabase/supabase-js) | v2 | auth + the `documents` table |
| [html2canvas](https://html2canvas.hertzen.com) | 1.4.1 | rasterizer fallback (lazily loaded) |
| [jsPDF](https://github.com/parallax/jsPDF) | 3.0.4 | PDF assembly (lazily loaded) |
| Google Fonts | Nunito 800, JetBrains Mono | editor + code aesthetics |
| your browser | — | the entire runtime (^.^) |

fonts for math glyphs in the suggestion popup: Cambria Math → STIX Two Math → Times, so the little `∑ ∫ ℝ` previews look right even before you commit.

---

## 🐾 known quirks

- **Safari** exports use the html2canvas path instead of SVG foreignObject, on purpose — Safari's foreignObject rendering is, charitably, interpretive art.
- the PNG/PDF exports re-render from the *text*, not the live DOM — so if a render error message is on screen, exports won't include it. features, not bugs.
- `Ctrl/Cmd+S` is PNG export *and* a cloud-save flush, so muscle memory from other editors produces pictures. sorry in advance (人´∀`)
- huge documents zoom smoothly because the preview is one transformed surface; huge *exports* auto-downgrade their scale to stay inside canvas limits rather than crash the tab.
- everything sync-adjacent is best-effort with retries: closing the tab mid-save still fires a `keepalive` POST (if the doc is under ~60 KB) and the `beforeunload` prompt asks you to stay a moment if a save is in flight.

---

## 📜 license

none attached — it's your file, treat it kindly. if you fork it into something great, that's just the circle of life ヾ(≧▽≦*)o
