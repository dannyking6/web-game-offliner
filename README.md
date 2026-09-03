# web-game-offliner 🎮

An agent skill for **Codebuff / any .agents-compatible agent**: download any 2D
web game from a portal (GameSnacks, Famobi, Softgames…), strip its platform SDK,
patch it into a **100% offline standalone build**, validate it with Playwright +
an application firewall, and deploy it to GitHub Pages with a clean ZIP release.

> Proven end-to-end on 5 shipped games (Construct 3, Phaser 2, Phaser 3,
> PixiJS 5, custom canvas engines) — including the traps: poisoned 404 assets,
> frozen ad callbacks, 2048px mobile GPU texture limits, minified-code surgery.

## What the skill does

```
Phase 0  IDENTIFY    catalog scan → engine fingerprint → 2D-only selection
Phase 1  DOWNLOAD    full asset pull + integrity audit (magic bytes!)
Phase 2  PATCH       SDK extraction → neutral game-driver.js
Phase 3  VALIDATE    Playwright + network firewall + mobile emulation
Phase 4  DEPLOY      GitHub repo + Pages + README + ZIP release v1.0.0
```

Every phase encodes the hard-won gotchas that break naive attempts:

- ✅ Ad callbacks (`adBreakDone`, `beforeReward`) **must fire** or the game freezes
- ✅ Sync vs async storage semantics matched to each wrapper (GameSnacks / Softgames / Famobi)
- ✅ 404-HTML-disguised-as-PNG detection via magic bytes
- ✅ Atlas resizing for mobile GPU 2048px texture limits
- ✅ Crash-dialog auto-recovery for transient WebGL/audio glitches
- ✅ Live-URL verification after Pages build, not just local testing

## Install

```bash
npx skills add dannyking6/web-game-offliner --yes
```

Or manually: copy `SKILL.md` into your project at
`.agents/skills/web-game-offliner/SKILL.md`.

## Usage

Just ask your agent:

> "Download Smarty Bubbles from GameSnacks and make it playable offline,
> then deploy it"

The agent loads the skill and follows the phased pipeline with all validation
gates.

## Shipped proof

| Game | Engine | Live |
|---|---|---|
| Guardians of Gold | Construct 3 | [play](https://dannyking6.github.io/guardians-of-gold/) |
| Geometry Rush | Phaser CE 2.15 | [play](https://dannyking6.github.io/geometry-rush/) |
| Jewels Blitz 5 | Phaser + Softgames wrapper | [play](https://dannyking6.github.io/jewels-blitz-5/) |
| Daily Word Search | PixiJS 5 + GSAP | [play](https://dannyking6.github.io/daily-word-search/) |
| Smarty Bubbles | Custom canvas + Famobi wrapper | [play](https://dannyking6.github.io/smarty-bubbles/) |

## License

MIT
