---
name: web-game-offliner
description: Download any 2D web game (GameSnacks, Famobi, Softgames, etc.), strip its platform SDK, patch it into a fully self-contained offline build with a neutral driver, validate it with Playwright plus an application-level firewall, then deploy it to GitHub Pages and publish a clean ZIP release. Use when the user asks to localize, offline-ify, self-host, mirror, de-SDK, or republish a browser game.
version: 1.0.0
author: buffy
tags: [games, offline, download, playwright, github-pages, game-snacks]
---

# Web Game Offliner

Take any 2D web game from a portal (GameSnacks, Famobi, Softgames, CrazyGames…)
and turn it into a **100% offline-playable standalone build** with zero external
dependencies, then deploy it. This skill encodes a pipeline proven end-to-end on
5 games (Construct 3, Phaser 2/3, PixiJS 5, custom canvas engines).

## Pipeline overview

```
Phase 0  IDENTIFY    → catalog scan, engine fingerprint, game selection
Phase 1  DOWNLOAD    → full asset pull + integrity audit
Phase 2  PATCH       → SDK extraction → neutral game-driver.js
Phase 3  VALIDATE    → Playwright + application firewall
Phase 4  DEPLOY      → GitHub repo + Pages + ZIP release v1.0.0
Phase 5  RETITLE     → AI picks a new short catchy name, generates a styled
                       title image, updates metadata, republishes repo+release
```

Never skip Phase 3: "it loads locally" is not "it works offline".

---

## Phase 0 — Identify the game and its engine

1. **Catalog scan** — GameSnacks' catalog is one big JSON payload. Fetch the
   homepage and pull every `/games/<id>` link (≈423 IDs). Each game page yields
   the **real CDN URL** hidden behind `h5games.usercontent.goog` plus its title.
2. **Engine fingerprint** — fetch each candidate's `index.html` and grep the
   runtime JS for:
   - `c3runtime` / `Scirra` → Construct 3
   - `Phaser` (check version, CE vs 3) → Phaser
   - `pixi.js` / `PIXI` → PixiJS
   - `cocos` → Cocos Creator
   - `BABYLON` → **disqualify (3D)**
   - `UNITY` / `UnityLoader` → **disqualify (not web-native)**
   - `godot` → **disqualify**
3. **Watch for false positives**: matching the substring `constructor` is NOT
   Construct; "Phaser" inside a comment is NOT Phaser. Confirm on the runtime
   file, not the index.
4. **Verify 2D-ness before committing**: probe for 3D markers (`BABYLON`,
   `webgl` 3D pipelines, camera.z). One candidate (Retro Drift) turned out to
   be Babylon.js despite an initial "Construct" match — it was disqualified.

Selection criteria: 2D only, web-native engine, total size under ~20 MB, clean
asset manifest, no aggressive DRM.

## Phase 1 — Download everything (and audit it)

1. **Enumerate assets from the manifest, not from guesses**:
   - Construct 3 → `data.json` (media list) + `appmanifest`
   - Phaser/PixiJS games → grep the loader calls in the game code
     (`load.image`, `load.atlas`, `load.audio`, preload manifests, `game.json`)
2. **Map the real path prefix** — assets often live under a subfolder
   (`assets/img_480/`, `assets/hd/`, `media/`). Sounding 404s with curl on the
   CDN *before* mass download saves an hour.
3. **Retina suffixes**: PixiJS loads may append `@2x` to names
   (`logo@2x.png`, `ui@2x.json`). Check for a resolution-suffix variable.
4. **Download all referenced files** (curl loop with a proper Referer header).
5. **MANDATORY integrity audit** — many traps produce corrupt files:
   - `file <each>` — anything reporting "HTML document" instead of
     PNG/JPEG/JSON/Audio is a poisoned file. This happens constantly (Google
     404 shells saved as `.png`, `.jpg`, even `.json`).
   - Every image must open with PIL/ImageMagick; every JSON must `json.load`.
   - Cross-check count and sizes against the manifest.
   - Delete fake files, retry with corrected paths, or replace with a clean
     transparent PNG / valid stub if the CDN itself 404s them (dead assets the
     original site also failed to load).
   - Fonts referenced by CSS must exist too (`gamefont.ttf` etc.).

## Phase 2 — Patch for offline play

### The core pattern: two-tier bridge + driver

Replace the platform SDK with a **neutral local driver**, never with dead
stubs:

```
index.html
  └── game-driver.js      ← NEW: neutral "driver" = offline SDK replacement
        └── exposes the exact surface the game calls
game code (js/game.js, main.js…)
  └── GameSnacks.* / famobi / sg calls rewritten to GameDriver.*
```

1. **Inventory the SDK surface actually used** — grep the whole codebase for
   every call shape: `GameSnacks.`, `window.GameSnacks`, `famobi.`, `sg.`
   (Softgames), `pSDK`. Typical methods: `audio.*`, `ad.*`, `storage.*`,
   `gameStart()`, `levelStart/levelEnd`, `happytime()`, `gameover()`,
   `setProgress`, `firstPlay`, etc.
2. **Write `game-driver.js`** implementing that surface faithfully:
   - `audio`: no-op or WebAudio-based stub (`sfx()`/`bgSound()` no-ops are
     fine, but keep the method names).
   - `ad`: the #1 source of frozen games. If the game passes callbacks
     (`beforeReward`, `adBreakDone`, `adViewed`…), **you MUST call them**
     (immediately with a benign status like `'dismissed'`). A no-op here
     freezes the game the moment it fires a "gameover" interstitial — this
     exact bug froze Daily Word Search at puzzle completion.
   - `storage`: replicate the game's expected semantics. Some wrappers expect
     synchronous `getItem/setItem` (JSON), others (Softgames) expect
     base64-encoded JSON values and **asynchronous** reads. Read the SDK
     client code to know which.
3. **Rewrite call sites** — in minified code, plain-text replacement works:
   `GameSnacks.` → `GameDriver.` (watch for `window.GameSnacks` too). If the
   game has a fallback stub like `window.GameSnacks = window.GameSnacks || {}`
   it's harmless — leave it.
4. **Neutralize analytics/telemetry**: Sentry, DDNA, Google Fonts, A/B test
   fetches, `img` social share paths. Either remove the script tags, or
   redirect to `about:blank`/local data URLs. **Careful with string surgery in
   minified code** — one broken `concat(...)` parenthesis cost an hour of
   `Unexpected token ')'`. Always `node --check` (or `node -e "import()"`)
   after editing JS.
5. **Neutralize sitelocks/domain checks** (grep `location.host`, `hostname`,
   whitelist arrays).
6. **index.html** — remove external `<script>`/`<link>` tags; only local
   files; add `game-driver.js` **first** in `<head>` (it must exist before any
   game script runs, and it's the right place for a global recovery hook).
7. **Keep a game fallback safe**: some games show "reload the page" on any
   uncaught error. Add an auto-recovery in the driver: watch for the crash
   dialog; if it appears, `location.reload()` once, guarded by a cooldown
   (30 s) so a persistent error doesn't loop.

## Phase 3 — Validate (this is what makes it trustworthy)

Run **all** of these; each has caught real bugs:

1. **Playwright pass** (Chromium, `--use-gl=swiftshader`):
   - canvas present and sized,
   - **0 page errors**, **0 failed requests**, **0 external requests**.
2. **Application firewall** — route-block every non-localhost request, then
   play the game (click through menus, start a level). The game must run with
   the network cut. This proves "no blocking external dependency".
3. **Real-browser conditions**, not just happy path:
   - mobile emulation (touch, mobile viewport, **no forced autoplay**) —
     audio unlock flows differ from headless defaults,
   - gameplay interaction: click where the core mechanic lives, then
     screenshot-diff across time to confirm real scene progression (static
     background + nothing = broken).
4. **Visual check**: dump screenshots to ASCII/color histograms; a grid of
   gem colors or a letter grid means the game truly rendered.
5. **Long-run probe**: let it idle 30–60 s; confirm the render loop steps
   (`game.loop.frame` / RAF counting) rather than a frozen first frame.
6. **Headless ≠ real GPU**: a 3646×3583 atlas rendered fine under SwiftShader
   but exceeded many mobile GPUs' 2048 texture limit (Jewels Blitz 5 showed
   only its background on a real phone). Audit texture sizes; if any atlas is
   > 2048 px, resize it and rescale the frame coords in its JSON atlas.
7. **Exercise the SDK paths you shimmed**: from the page, call the driver's
   ad-break flow (reward + interstitial) and assert the callbacks resolve in
   ms. This catches the frozen-callback class of bugs before deploy.

## Phase 4 — Deploy

1. **ZIP release**: archive ONLY the game files (no `.git`, no tooling), named
   `<game>-v1.0.0.zip`; verify zero `.git` entries inside.
2. **GitHub repo** (one per game), push, enable **Pages via API**
   (`gh api repos/<owner>/<repo>/pages -X POST`), wait for the Pages build
   (poll the API until `status: built`).
3. **Live verification**: Playwright against the GitHub Pages URL — 0 errors,
   0 third-party requests, game advances past menus.
4. **README.md** with: flattering game description, **controls table per
   device** (smartphone/tablet tap, desktop left-click, trackpad single click,
   keyboard not required), offline modifications list, run instructions
   (`python3 -m http.server`).
5. **serve.sh**: tiny script `python3 -m http.server "$PORT" --bind 0.0.0.0`
   (games need HTTP; `file://` won't work).
6. **Release v1.0.0** with the ZIP as asset via `gh release create`.

## Phase 5 — Retitle the game (MANDATORY for every published game)

Every deployed game gets a **new original title**: never ship the portal's
original branding. **The AI agent must invent the name itself** — pick a short
(2 words max), catchy, genre-fitting name (e.g. "Frosty Rush" for a snowman
puzzle game, "Pocket Golf" for a mini-golf game). Do not ask the user to name
it; propose it in the final report.

1. **Find the title asset** (one of):
   - dedicated logo file (`logo_main`, `game-logo.png`, `title.png`…)
   - a frame inside a texture atlas (grep the atlas JSON for `title`/`logo`)
   - a font-rendered Text object in the scene data (then edit the string + font)
2. **Generate the title image** with PIL — requirements learned the hard way:
   - **Size the text with stroke metrics included**: `textbbox(..., stroke_width=n)`
     WITHOUT the stroke understates width by ~20px/char and the title gets
     clipped at the edges (a real bug that hid the P and F of a title).
   - **Advance letters with `font.getlength()`**, never with per-letter bbox
     sums — per-letter bbox widths overflow the measured string bbox.
   - Auto-fit with a binary search on font size against `canvas_w - 2*margin`;
     assert the alpha bbox stays inside the canvas before saving.
   - Style: per-letter color palette or vertical gradient (NOT plain white),
     dark outline, blurred drop shadow, gloss highlight on the top half.
     Compose layers with `Image.alpha_composite` — `ImageDraw` on RGBA
     REPLACES pixels instead of blending (a fill with alpha 0 erased a whole
     gradient once).
   - Render at 2× then LANCZOS-downscale to the exact original sprite dims.
3. **Patch the engine**:
   - Construct 3: replace the PNG frames, keep `data.js` untouched (dims must match).
   - Phaser/PixiJS atlas: either redraw the frame region in the atlas PNG, or
     (cleaner) add a standalone image, `load.image()` it in the loader, and
     repoint the title object (`this.titleImg = ...`) to the new texture key.
4. **Update all metadata**: `<title>` + `meta[name=application-name]` in
   index.html, PWA manifest `name`/`short_name`, `GameData.BuildTitle`-style
   constants in game.js, i18n strings that embed the old name.
5. **Verify** the full title renders (pixel-scan the title zone for the palette
   colors reaching both left and right edges) and 0 console errors.
6. **Republish EVERYTHING**: push to the repo (Pages rebuilds), wait for the
   CDN to serve the new files, AND **create a new release** (v1.0.1, v1.0.2…)
   with a fresh ZIP — existing GitHub releases are immutable, so a retitle is
   never done until a new release carries the new build.

## Hard-won gotchas (read before debugging)

- **404-HTML-as-asset** is everywhere: a "successful" download can be a Google
  error page. Verify file magic bytes, not just HTTP 200.
- **Ad callbacks must fire** or the game freezes at the next interstitial.
- **Match the storage semantics exactly** (sync vs async, base64 vs plain).
- **Texture size limits** (2048) on real mobile GPUs; headless SwiftShader
  hides the bug.
- **Dead CDN assets** (404 on origin too): replace with valid stubs instead of
  shipping error-HTML files.
- **Minified-code edits**: one wrong character kills the whole bundle. Prefer
  whole-string replacements, verify with `node --check` after every patch.
- **node resolves modules from the script's directory, not cwd** — keep test
  scripts in the folder where playwright is installed.
- **CDN/Pages caching** after push: poll until the live content hash changes
  before re-testing.
- **Autoplay policy**: always launch tests with
  `--autoplay-policy=no-user-gesture-required` but ALSO test without it under
  mobile emulation.

## Reference implementation

A complete working example of every artifact this skill describes (driver
files, firewall tests, atlas resizing, deploy scripts) lives in
`gamesnacks-batch/*/game-driver.js` and `gamesnacks-local/` in the agent
workspace that produced it. Reproduce the same artifact names so future agents
can navigate quickly: `game-driver.js`, `serve.sh`, `README.md`,
`resize_atlas.js`, `test_*.js`.
