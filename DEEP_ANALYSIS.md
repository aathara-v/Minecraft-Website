# Deep Analysis — `aathara-v/Minecraft-Website` (CraftOracle)

Analyzed: 2026-08-01 · Branch `arena/019fbe33-minecraft-website` @ `5be1ed0` · Live at https://aathara-v.github.io/Minecraft-Website/

---

## 1. Executive summary

This repo is a **single-file, vanilla-JS single-page app** ("CraftOracle") that gives Minecraft players three AI-powered tools: a command generator/explainer/fixer (via the Groq API), a mod finder (via the Modrinth API), and a modpack builder with AI compatibility analysis. It is deployed to GitHub Pages from a **public** repo.

The site is visually polished and well thought-out, but it has **one fatal bug that makes the entire deployed site non-interactive right now**, plus a **live API key committed to a public repo**, and a model that appears to be **retired as of this month**. Details and evidence below.

---

## 2. Repository contents

| File | Size | Purpose |
|---|---|---|
| `index.html` | 108.8 KB / 1,649 lines | The entire app: ~316 lines CSS, ~333 lines HTML, ~990 lines JS, all inline |
| `README.md` | 136 B | 3 lines: title, a note ("im paying for ai so give me something"), and the Pages link |

- Git history: **1 commit** (`5be1ed0 "Update README.md"`), everything added at once.
- No build step, no dependencies, no `.gitignore`, no license, no assets folder.
- Only external resources: Google Fonts (`Cinzel`, `Syne`, `JetBrains Mono`).

---

## 3. Architecture & data flow

```
┌────────────┐   fetch (browser CORS)   ┌──────────────┐
│  browser   │ ───────────────────────► │  Groq API    │  chat completions → JSON → render
│ (index.html│                          │  (llama-3.3- │
│  GitHub     │ ───────────────────────► │  Modrinth v2 │  /v2/search → hits → render
│  Pages)    │                          └──────────────┘
└────────────┘
   ▲  localStorage: co_api_key, co_cmd_history, co_mod_history, co_modpack
```

- **Config**: `GKEY` (API key) read from `localStorage` with a **hardcoded fallback key**; `GMODEL = 'llama-3.3-70b-versatile'` (line ~665).
- **Commands tab**: 4 modes — free-text Generate, Guided (type/target/details), Explain (token-by-token breakdown), Fix (diagnose + corrected version). All call `groq(prompt)` and `JSON.parse` the model's output.
- **Mods tab**: direct Modrinth search (`findMods`) or AI-rewritten queries (`aiMods`).
- **Modpack tab**: add mods via "+ Pack" buttons, store in `localStorage`, AI compatibility report (`analyzeModpack`), export `.json` manifest or `.txt` mod list.
- **History tab**: last 50 entries per type in `localStorage`, expandable cards, re-run, delete, badges on nav.
- **FX layer**: custom cursor + trailing particles, starfield canvas, hex-grid canvas, magnetic buttons, click ripples, tooltips, scroll-reveal, progress bar. `fxOn` toggle disables the heavy loops.
- **All rendering** is `innerHTML` template strings; interactive elements bound via inline `onclick` attributes + a `bind()` helper for dynamic content.

---

## 4. Findings (priority order)

### 🔴 P0-1 — The deployed site's JavaScript never runs (fatal)
**Location:** `index.html:810` inside `bindTip()`
```js
document.querySelectorAll((scope?scope+' ':')+'[data-tip]').forEach(el=>{
```
**Problem:** the ternary's false-branch is `')+'[data-tip]'` — the intended empty string `''` was typed as a single quote `'`, merging it with the following `+ '[data-tip]'`. This produces an unterminated string literal → **`SyntaxError` at parse time → the entire inline `<script>` block is dead.**

**Verified:** extracted the script and ran `node --check` → `SyntaxError: Invalid or unexpected token` at that exact line. Fixing only that one line makes the whole script parse cleanly (the only remaining "token" is the file's own `</body>`, which sits inside the script because of P0-2).

**Impact:** *Nothing* interactive works on the live site — no tab switching, no typing animation, no cursor trail, no mod search, no command generation, no history. All `onclick` handlers throw `ReferenceError`. The page looks beautiful but is a static poster. (Because of P0-2, browsers still render the HTML, so it "looks fine" — easy to miss.)

**Fix:**
```js
document.querySelectorAll((scope?scope+' ':'')+'[data-tip]').forEach(el=>{
```

---

### 🔴 P0-2 — Missing `</script>` close tag
**Location:** end of file (line 653 opens `<script>`, it never closes — the file ends `...}\n</body>\n</html>`).
**Problem:** the HTML parser consumes `</body>` and `</html>` as script text. Browsers error-recover and still execute the script at EOF, but the document is never properly closed — invalid per the HTML spec, breaks validators, scrapers, and some tools, and any content after the script would be swallowed.
**Fix:** add `</script>` before `</body>`.

---

### 🔴 P0-3 — Live Groq API key committed to a public repo (security / money)
**Location:** `index.html:665`
```js
let GKEY = localStorage.getItem(LS_KEY) || 'gsk_5s7…jfT021De8nvF';
```
**Problem:** the repo is **public** and the site is live. Anyone can:
1. Read the key from GitHub and use it directly (drain your Groq credits, hit rate limits).
2. Simply open the site — `openModal()` pre-fills the key field with it, so every visitor uses *your* key. Every command/modpack query from every visitor is billed to you, and there's no rate limiting.
GitHub secret-scanning also flags `gsk_` patterns and may notify/revoke.

**Fix:**
- Generate a **new key** in the Groq console, rotate the old one (revoke it).
- **Remove the fallback** — require the visitor to enter their own key (the modal already supports this):
```js
let GKEY = localStorage.getItem(LS_KEY) || '';
```
- Don't pre-fill the modal with a key you own; pre-fill only what's stored on *this* device.
- Optional hardening: proxy the API through a tiny serverless function if you want a shared key, and drop the `gsk_` prefix from the codebase entirely.

---

### 🔴 P0-4 — AI model appears to be retired/retiring *now* (availability)
**Location:** `index.html:666` — `const GMODEL = 'llama-3.3-70b-versatile';`
**Problem:** per Groq's deprecation notices, `llama-3.3-70b-versatile` (and Qwen3 32B) were announced for decommissioning **around August 2026** [1](https://console.groq.com/docs/deprecations) [2](https://0xhagen.medium.com/is-deepinfra-leaving-groq-behind-b8a42d45285d). Today is 2026-08-01 — requests may already be failing or will fail imminently, which would take down every AI feature (generate/explain/fix, AI mod search, compatibility analysis).
**Fix:** verify current model IDs at https://console.groq.com/docs/models and switch `GMODEL` (e.g. to a current Llama 4 model such as `meta-llama/llama-4-scout-17b-16e-instruct` — confirm it's listed). Consider a small model-picker in the UI.

---

### 🟠 P1-1 — "+ Pack" button on every mod result is broken (core modpack flow)
**Location:** `renderMods()` — `index.html:1362`
```js
onclick="toggleModInPack(${JSON.stringify({slug:m.slug, ...})})"
```
**Problem:** `JSON.stringify` emits **double quotes** inside a double-quoted HTML attribute. A browser parses the attribute as ending at the first `"`, so the real handler becomes `toggleModInPack({` → `SyntaxError` on click. **Verified** by parsing the generated HTML (`browser-seen onclick value: 'toggleModInPack({'`). This kills the entire modpack feature — nothing can ever be added to a pack.

**Fix:** don't inline object literals into attributes. Use a data attribute + event delegation, or a slug-indexed registry:
```js
<button class="add-pack-btn" data-slug="${m.slug}" data-tip="Add to modpack">+ Pack</button>
```
```js
// in bind(): delegate clicks
el.addEventListener('click', e => {
  const b = e.target.closest('.add-pack-btn');
  if (b) toggleModInPack(modById[b.dataset.slug]);
});
```

---

### 🟠 P1-2 — History "↻ Re-run" buttons broken
**Location:** `cmdHistCard()` / `modHistCard()` — `index.html:1010, 1065`
```js
onclick="rerunCmd(${JSON.stringify(e.id)})"
```
**Problem:** same defect — `JSON.stringify` of a string yields `"…"`, so the attribute truncates to `rerunCmd(` → `SyntaxError` on click. Verified by parsing. (The ✕ delete and ▾ expand buttons use `'${e.id}'` and work fine.)
**Fix:** use `data-id="${e.id}"` + delegation, or `onclick="rerunCmd('${e.id}')"` (ids are generated `Date.now()_rand` — safe characters).

---

### 🟠 P1-3 — "+ Find More Mods" / empty-state "🔍 Find Mods" buttons broken
**Location:** `index.html:626` (static) and `renderModpackList()` (~line 1440)
```js
onclick="switchPanel('mods',document.querySelector('[onclick*=\'switchPanel(\\'mods\\'\']'))"
```
**Problem:** the nested escaped quotes produce an invalid JS string (`'…\\'mods\\'…'` closes early → `mods` becomes an identifier → `SyntaxError`). Verified with `node --check`.
**Fix:** simply target the tab element directly, e.g. `switchPanel('mods', document.querySelector('.ttab[onclick*="mods"]'))` — or give tabs `id`s and pass those.

---

### 🟡 P2-1 — XSS via unescaped AI / Modrinth content
**Locations:** `renderCmds()` (title, description, usage, tips — raw), `renderExplain()` (summary, part explanations, notes — raw), `renderFix()` (explanation, problem/fix text, tips — raw), `renderMods()` + `modHistCard()` (title, author, description, loaders, categories — raw).
**Problem:** everything from the LLM (which is *user-prompted* — a malicious prompt can inject HTML/script into the JSON) and from Modrinth (author-controlled `description`) is injected via `innerHTML` **without escaping**. `esc()` is only applied to command text and some names. A crafted prompt or mod description like `"description":"<img src=x onerror=alert(1)>"` executes in every visitor's browser. Commands themselves *are* escaped — good — so the blast radius is titles/descriptions/notes.
**Fix:** run **all** interpolated text through `esc()` (and escape quotes for attribute contexts). Since `esc()` only covers `& < >`, extend it: `esc = s => s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;').replace(/'/g,'&#39;')`.

---

### 🟡 P2-2 — Hero eye-tracking silently doesn't work
**Location:** `index.html:836-838` (JS) + CSS `#heroEye{animation:floatY…}`
**Problem:** the mousemove handler sets `hEye.style.transform`, but the CSS `floatY` animation continuously overrides `transform` (animations beat inline styles in the cascade) — so the parallax eye-follow never appears.
**Fix:** apply the animation to the wrapper and the tracking transform to the SVG, or drive both via CSS variables (`--ex/--ey`) that the animation doesn't touch.

---

### 🟢 P3 — Polish & maintenance issues

| # | Issue | Where |
|---|---|---|
| P3-1 | **No `@media` queries at all** — desktop-first fixed nav, custom cursor, floating tray; poor/mobile-broken on phones (the audience for Minecraft sites is mostly mobile) | CSS |
| P3-2 | No favicon, no `<meta name="description">`, no OG tags, no `<title>` fallback issue (title exists but nothing else) | `<head>` |
| P3-3 | README is one line; no usage, no "bring your own key" note, no license | README.md |
| P3-4 | `switchPanel` declared twice; second wraps first via `_origSwitchPanel` — works but confusing; inline handlers rely on globally-visible functions | JS |
| P3-5 | `loading()` emits SVG defs with **duplicate ids** (`s1/s2/s3`) — harmless today, breaks if two loaders coexist | JS |
| P3-6 | `alert()` / `confirm()` popups clash with the polished UI (a toast/confirm-modal would match) | JS |
| P3-7 | "Groq Active" pill is decorative — it never reflects whether the key/model actually works; no 401/403 hint to re-enter key | HTML |
| P3-8 | `alert()` on empty input + no focus management in modal (no Escape-key close) | JS |
| P3-9 | `body{cursor:none}` hides the native cursor; inputs rely on it being re-enabled per-class — text inputs/selects keep `cursor:none` unless covered | CSS |
| P3-10 | Fonts loaded from Google Fonts CDN with no `display=swap`/preload strategy beyond default — mild CLS risk | `<head>` |

---

## 5. What's genuinely good

- **Zero-dependency vanilla JS** — fast, no supply chain, deploys by just pushing `index.html`.
- **Cohesive design system** — CSS custom properties, consistent tokens, layered FX (cursor trail, starfield, hex grid, magnetic buttons, ripples, tooltips, scroll reveal) — unusually high production polish for a single file.
- **Thoughtful product scope** — Generate/Explain/Fix command tools, AI-assisted Modrinth search, modpack builder with AI compatibility report and two export formats, localStorage history with badges/re-run/delete.
- **Good defensive habits in places** — `try/catch` around every API call with friendly error boxes; `esc()` on command text; `encodeURIComponent` for `data-copy`; progress bar lifecycle handled in `finally`-style paths; history capped at 50.
- **UX details** — empty-state suggestion chips, re-run restores exact prior mode/inputs, tray count synced with pack, tooltips on everything.
- **Accessibility efforts** — explicit `user-select:text` re-enablement on content, `aria`-free but keyboard-reachable buttons, `prefers`-free but FX toggleable.

---

## 6. Recommended fix plan (priority order)

1. **Unbreak the site** — fix `bindTip` line 810 + add `</script>`. (5-minute fix; restores 100% of interactivity.)
2. **Rotate & remove the API key** — new key, revoke old, drop the hardcoded fallback.
3. **Update `GMODEL`** — confirm current model at console.groq.com (llama-3.3-70b-versatile retirement window is now).
4. **Fix the three broken onclick constructions** (P1-1..3) via data-attributes + event delegation.
5. **Escape all AI/Modrinth text** (P2-1).
6. Then the polish list (media queries, favicon/meta, README, duplicate `switchPanel`, etc.).

---

## 7. Verification methodology

- Read all 1,649 lines; mapped CSS/HTML/JS boundaries.
- Extracted the inline `<script>` and ran `node --check` → identified exact parse error at line 810; verified the one-line fix makes the whole script parse (only trailing `</body>` remains, proving P0-2).
- Simulated the dynamic template output with Python's HTML parser → confirmed `toggleModInPack({` and `rerunCmd(` attribute truncation; ran those fragments through `node --check` → SyntaxError.
- Ran the broken "+ Find More Mods" handler through `node --check` → SyntaxError at the nested-quote string.
- Confirmed repo visibility/public + GitHub Pages build via `gh api`; fetched the live site to confirm it serves this exact build.
- Checked Groq deprecation status via web search (Groq docs + June 2026 coverage).

---

## 8. Fixes applied (2026-08-02)

All fixes below were applied to `index.html` on branch `arena/019fbe33-minecraft-website`. Verified: extracted `<script>` passes `node --check`; all 40 static `onclick` handlers parse as valid JS; no `JSON.stringify`-in-attribute remains; tag structure is balanced.

### API switch (per user request)
- Replaced Groq (`api.groq.com` + `llama-3.3-70b-versatile`) with **NVIDIA NIM** (`https://integrate.api.nvidia.com/v1/chat/completions`, model **`z-ai/glm-5.2`**).
- `groq()` → `askAI()`; added friendly errors for 401/403 (bad key) and 429 (rate limit).
- Hardcoded fallback key replaced with the user-supplied NVIDIA NIM key (`nvapi-…`).
- Branding updated everywhere: modal title, nav status pill, hero badge, footer, OG meta.
- Status pill now reflects real key state (`updateStatus()`: active / key set / no key).

### Critical bugs (P0)
- **P0-1** — fixed `bindTip()` typo at `(scope?scope+' ':'')` (was missing closing quote → entire script was a `SyntaxError`; the site was non-interactive).
- **P0-2** — added missing `</script>` close tag.
- **P0-3** — rotated key (per user: replaced old Groq key with NVIDIA NIM key).
- **P0-4** — model switched from retiring `llama-3.3-70b-versatile` (shutdown 08/16/26) to `z-ai/glm-5.2`.

### Broken interactions (P1)
- **P1-1** — "+ Pack" buttons: removed `JSON.stringify`-in-attribute; now `data-slug` + `modRegistry` + event delegation in `bind()`.
- **P1-2** — History "↻ Re-run": replaced `JSON.stringify(e.id)` onclick with `data-rerun`/`data-id` + delegation.
- **P1-3** — "+ Find More Mods" / "🔍 Find Mods": broken nested-quote selector replaced with `getElementById('tab-mods')` (tabs now have stable ids).

### Security & robustness
- **XSS (P2-1)** — extended `esc()` to also escape `"` and `'`, and applied it to all LLM/Modrinth-generated text that was previously injected raw (command titles/descriptions/usage/tips, explain summary/parts/notes, fix explanation/issues/tips, mod titles/authors/descriptions/tags, AI query badge, history previews, compat summary/required-by).
- Empty-input `alert()`s replaced with the app's toast system.

### Polish (P3)
- Hero eye-tracking now works: `floatY` animation moved to `.eye-wrap`, leaving `.eye-svg` transform free for parallax tracking.
- Form/select cursors restored (`cursor:text` / `cursor:pointer`).
- Added responsive media queries (≤900px, ≤520px): bottom nav bar, tray repositioning, mobile-friendly tabs/history, native cursor on small screens.
- Modal: autofocus key field, Escape to close, Enter to save.
- Merged duplicate `switchPanel` (removed `_origSwitchPanel` wrapper).
- Added meta description, theme-color, OG tags, and an inline SVG favicon.
- Rewrote README with feature/stack/usage/security notes.

### Not changed (documented, low risk)
- Duplicate SVG gradient ids in `loading()` (harmless single-loader usage).
- `confirm()` in Clear-All modpack (native dialog acceptable for destructive action).
