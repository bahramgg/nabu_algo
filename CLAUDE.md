# CLAUDE.md

Guidance for AI assistants (and humans) working in this repository.

## What this is

**Algo Mirror** — a single-page web app that audits an X (Twitter) account and
scores how the X recommendation algorithm likely "sees" it. The user enters a
handle, the app pulls the account's profile + recent tweets, runs a blend of
deterministic client-side scoring and an LLM content/risk analysis, then renders
an overall score, archetype, radar chart, factor breakdown, and shareable card.

Live site: **algo-nabulines.com** (see `CNAME`), hosted on GitHub Pages.

## Repository layout

This is intentionally tiny — there is no build system, framework, or package manager.

```
index.html   The entire application: markup + inline <style> + inline <script>
CNAME        GitHub Pages custom domain (algo-nabulines.com)
```

That's it. **All code lives in `index.html`** (~986 lines). The file is ~164 KB
mostly because two logo images are embedded as base64 data URIs near the top of
the script (`LOGO_WHITE` / `LOGO_BLACK`, lines ~344–345). Those two lines are
enormous — when reading or searching the file, target specific line ranges or
use `Grep` rather than reading the whole file, and don't try to reformat the
base64 blobs.

## Architecture of `index.html`

The single file has three parts:

1. **`<style>`** (top of `<head>`) — all CSS. Theming is driven by CSS custom
   properties on `:root` and `[data-theme="light"]`. Dark is the default.
2. **HTML body** — three stacked screens toggled by class/`display`:
   - `#splash` — intro screen, dismissed by `goApp()`.
   - `#app` → `#heroSection` — the handle input (`#usernameInp`) + analyze button.
   - `#loadingDiv` — stepped 16s loading animation.
   - `#resultsDiv` — results, rendered into by `renderResults()`.
3. **`<script>`** (bottom of `<body>`) — all logic, plain vanilla JS, no modules,
   no bundler. Runs on load via `applyLogos()` and `initReveal()`.

### Data flow (the `analyze()` function, ~line 494)

Everything funnels through one `PROXY` constant (line 343) — a Cloudflare Worker
that fronts all third-party calls and holds the API keys (keys are **never** in
this repo). The worker is addressed by an `?action=` query param:

| Action            | Purpose                                                                 |
| ----------------- | ----------------------------------------------------------------------- |
| `?action=twitter` | Proxies `twitterapi.io` (user info + `last_tweets`) via `proxyFetch()`. |
| `?action=openrouter` | Proxies OpenRouter chat completions for the AI analysis.             |
| `?action=getcache` / `?action=setcache` | Reads/writes a KV result cache keyed by lowercased handle (48h TTL, change-detected via a `sig` signature). |

The pipeline:
1. Check KV cache (`getcache`); if the cached `sig` matches current activity, reuse it.
2. Fetch profile + last 200 tweets (filtered to the last 90 days).
3. Compute deterministic factor scores client-side:
   - `scoreEngagement()` — follow conversion, reply depth, RT/quote power, media, etc.
   - `scoreStructural()` — posting frequency, originality, follower ratio, maturity, link penalty.
   - `scoreEligibility()` — premium boost, freshness, activity.
4. Call the LLM (`buildPrompt()`) for **content** and **risk** factors plus
   `archetype`, `oneLineProfile`, and `recommendations`. Model: `anthropic/claude-haiku-4.5`.
   The prompt encodes a strict "X algorithm auditor" scoring doctrine and expects
   JSON-only output.
5. Calibrate: AI scores are deliberately compressed (`squash`), then `calcOverall()`
   blends categories by weight, and an "algorithm reality model" (originality,
   conversation, depth, TweepCred-style authority, spam-farm penalties) adjusts the
   final number. Retweet-heavy accounts are hard-capped.
6. Cache the result (`setcache`) and `renderResults()`.

### Scoring conventions (important — easy to get backwards)

- All factor scores are **0–100, higher = better**, *except* the **risk** factors
  (`Block Risk`, `Mute Risk`, `Report Risk`, `Not Interested`) where **higher =
  safer**. Keep this inversion in mind when changing prompts or calibration.
- Category weights live in `calcOverall()` (`w = {engagement:.40, content:.22,
  risk:.13, structural:.15, eligibility:.10}`).
- The calibration curves and the "reality model" in `analyze()` are deliberately
  harsh/tuned. If you change a scoring formula, expect the visible overall score
  to shift — don't "fix" one without understanding the downstream multipliers.

### Rendering helpers

`renderResults()` orchestrates the results screen. Notable pieces: `animateScore()`
(count-up ring), `drawRadar()` (SVG radar), `renderGrid()`/`renderCatBars()` (factor
cards), `TIPS` object (factor explanation tooltips, ~line 348), `downloadCard()` /
`drawCardRadar()` (canvas share-card export), and `shareToX()`. The animated
background is a self-invoking canvas IIFE at the bottom (`initReveal()` / `draw()`).

## Development workflow

- **No build, no install, no tests.** Edit `index.html` directly.
- To preview locally, open the file in a browser or serve the folder, e.g.
  `python3 -m http.server` then visit `http://localhost:8000`. Note: live data
  requires the Cloudflare Worker `PROXY` to be reachable; without it, analysis
  calls fail but the splash/UI still render.
- **Deployment is automatic via GitHub Pages** from the default branch — merging
  to `main` publishes to algo-nabulines.com. There is no CI/CD config in the repo.
- Don't add a `package.json`, framework, or bundler unless explicitly asked; the
  project's whole premise is a single self-contained HTML file.

## Conventions to follow

- Keep everything in `index.html` — inline CSS in `<style>`, inline JS in `<script>`.
- Match the existing terse, dense code style (short variable names, single-line
  helpers, minimal comments — comments are reserved for explaining the *why* of
  calibration choices). Don't refactor toward modules/classes.
- Reuse the CSS custom properties for any new colors so light/dark theming keeps
  working.
- Route any new third-party call through the `PROXY` worker with a new `?action=`;
  never embed API keys, tokens, or secrets in this file.
- Preserve the score-direction conventions above and the JSON contract between
  `buildPrompt()` and the parsing in `analyze()` (the code slices the first `{`
  to last `}` and `JSON.parse`s it — the model must return valid JSON).

## Git / branch notes

- Default branch: `main` (this is what GitHub Pages publishes).
- The history here is mostly "Update index.html" commits — the working pattern is
  to edit the one file and commit with a short descriptive message.
- Do not create pull requests unless explicitly asked.
