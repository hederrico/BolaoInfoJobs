# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Bolão MELI** — a single-page web app (UI in Brazilian Portuguese) for a World Cup 2026 prediction pool. A fixed group of friends predicts scorelines; the app shows live scores, scores each prediction, and ranks players in real time.

The entire app is one file: [index.html](index.html) — HTML + CSS in `<style>` + vanilla JS in `<script>`. There is **no build step, no framework, no package manager, no tests, no lint config.** Dependencies are loaded from CDNs at runtime (Supabase JS, Google Fonts, flag images).

## Running / deploying

- **Run:** open `index.html` in a browser, or serve the folder statically (e.g. `python -m http.server`). A network connection is required — the app fetches the live feed, Supabase, fonts, and flag images from external services.
- **Deploy:** commit and push to `origin` (GitHub: `hederrico/BolaoInfoJobs`). There is no CI/build; the repo is the deployable artifact.

## Architecture (the non-obvious parts)

### Three layers of data, with fallback
1. **Live match feed** — `GET https://worldcup26.ir/get/games` (public, no auth), polled every 30s via `tick()` → `fetchGames()`. Populates the global `GAMES` array. The feed carries the **whole tournament** (104 games). Key per-game fields: `type` (`group` | `r32` | `r16` | `qf` | `sf` | `final` | `third`), `group` (`A`–`L` for group games), `matchday`, `home_team_id`/`away_team_id` (numeric string; **`'0'` = team not yet decided** in knockout), `home_team_label`/`away_team_label` (knockout source, e.g. "Winner Match 74" / "Winner Group E" / "3rd Group A/B/C/D/F"), `home_score`/`away_score`, `finished` (`"TRUE"`), `time_elapsed`, `local_date`, `stadium_id`. The `/get/stadiums` endpoint gives each `stadium_id`'s city/country/region.
2. **Shared predictions** — Supabase table `palpites` (`game_id, player, home, away`, upsert `onConflict: "game_id,player"`). `supaPull()` reads all rows into `GUESS`; `supaUpsert()` writes on each keystroke. URL + anon key are hardcoded near the top of the script.
3. **Local fallback** — if Supabase is unavailable, predictions persist to `localStorage` under key `bolao_meli_live_v1` (`saveG`/`loadG`), and if even that fails, to the in-memory object `mem`.

`GUESS` is the single source of truth for predictions, shaped as `{ gameId: { player: {a, b} } }` where `a`/`b` are home/away scores as strings (`""` = no prediction).

### Scoring model (`pts()`)
- **Cravou** (exact score): +6 · **Vencedor/empate** (correct outcome): +3 · **Errou**: 0 · no result yet: `pend`.
- `BASELINE` holds each player's points accumulated **before** this app existed; `buildGeral()` sums `BASELINE + finished games + live games` to produce the live standings. `PLAYERS` is the fixed roster — both are hardcoded constants.

### Tabbed views & the standings/scenarios engine
The app is split into four tabs (`activeTab`, built by `buildTabs()`, dispatched by `renderActive()`):
- **Bolão** (`render()`) — the original prediction/scoring view (live cards, próximos, geral, histórico). Live group game cards also embed a live mini-standings table.
- **Grupos** (`renderGroups()`) — standings tables for all 12 groups (recomputed live) plus a **best-thirds** ranking (`thirdsRanking()`: the 12 third-placed teams ordered points→GD→GF, top 8 qualify). The qualifying thirds get a "Classificando" badge in their group table (`qualThirdSet()`).
- **Mata-mata** (`renderBracket()`) — vertical knockout split into the two halves of the draw ("lado de cima"/"lado de baixo"). The bracket tree is reconstructed from the feed: each knockout game's `home_team_label`/`away_team_label` names its source ("Winner Match 74", "Winner Group E", "3rd Group A/B/C/D/F", "Loser Match 101"), and the game `id` IS the official FIFA match number (group 1–72, r32 73–88, r16 89–96, qf 97–100, sf 101–102, third 103, final 104). `koTree()` walks the source labels from the two SFs (M101/M102) to compute each half's match set and a DFS order that keeps bracket siblings adjacent. The view is a **horizontal connected bracket**: round columns (16-avos→Final) laid out with `justify-content:space-around`, and an absolutely-positioned SVG overlay (`drawBracketLines()`) draws the elbow connectors by measuring each card's real position (`getBoundingClientRect`) and linking each match to its two source matches — robust to variable card heights. Each card shows its number + both sources (`ptKoLabel()` translates them), so undecided slots still show who-plays-whom (e.g., "Venc. Jogo 74"). Horizontal scroll on `.bkscroll`; 3rd-place sits below the tree.
- **Cenários** (`renderScenarios()`) — per-group qualification scenarios. The same compact scenario is also embedded (collapsible) inside live/upcoming game cards on the Bolão tab via `cardScenarioSection()`.

Icons are inline SVG (`ICONS`/`icon()`), not emoji. The scenario grid is `table-layout:fixed` so it fits mobile width; outcome cells show only a flag (winner) or `=` (draw). The current scenario (matching the live/now scores) gets a gold outline on its whole `<tr>` (`s.cur`, computed in `scenarioFor`).

### Engagement features (all reuse the scenario engine)
- **Status selo** (`teamStatus()`/`statusBadge()`) — per team in group tables: green ✓ *Classificado* (guaranteed top-2 = `allPos⊆{1,2}`), red ✗ *Eliminado* (`allPos={4}` / finished 4th), gold ✓ *Classificando* (provisional best-third). Best-third status is intentionally "provisional" while other groups are unfinished (cross-group, not guaranteed).
- **"O que está em jogo"** (`stakesFor()`/`stakeChipsHTML()`) — for each of the two teams, the possible finishing positions if THIS game ends win/draw/loss (marginalizes the other remaining games via full scoreline enumeration). Cached in `stkCache` by `groupSig`. On **live cards** the three `V`/`E`/`P` chips render **directly under each team's name inside the scoreline** (`.team .stkmini`, color-coded ok/mid/out via `posClass`); on **upcoming cards** the legacy one-line-per-team block (`buildStakes()`) stays in the collapsed body.
- **Share / print** — canvas builders, all CORS-safe via **`crossOrigin="anonymous"` flags** (flagcdn.com sends `Access-Control-Allow-Origin:*`, so drawing flags does **not** taint the canvas — verified). `loadFlags(ids)` preloads (cached in `_flagImg`) and returns a `id→Image` getter; `drawFlag()` clips a rounded cover-fit flag; shared chrome via `cvBase`/`cvBrand`/`cvFooter`/`cvLegend`. Builders (all `async` except the flag-free ranking): `makeShareCanvas()` (ranking, statusbar + Geral panel), `makeMatchCanvas(g)` (one match: flags, **white** score, each player's prediction + points colored to match the DOM panel via `PTCOL` — cravou green / vencedor gold / errou gray, **no action-word text** — plus the live group mini-standings "como está o grupo"), `makeGroupCanvas(L)` (full group table + status dots, Grupos tab), `makeThirdsCanvas()` (best thirds, Grupos tab), `makeScenarioCanvas(L)` (Cenários tab — `scenarioGridCanvas`/`scenarioAggCanvas`/final standing). All go through `openShareModal(builder, filename, title)` where **builder may be a canvas or a (sync/async) function returning one**; it shows the `#shmodal` preview (`max-height:90vh`, scrollable, spinner while building) so the result is always visible, then offers **Baixar imagem** (`doDownloadBlob`, always) and **Compartilhar** (`doShareBlob` → `navigator.share({files})`, only when `navigator.canShare` supports files). Per-panel `.cardshare` buttons live in each live/history card, the Geral panel, every group + thirds card (Grupos), and every scenario card (Cenários).
- **Pull-to-refresh** — touch handlers (bottom IIFE) drive the `#ptr` indicator and call `tick(false)`.

The shared engine: `tableFromResults()` builds a table from a result list; `orderGroup()` sorts by **points → head-to-head mini-league → overall GD → overall GF** (note: this is H2H-before-GD, matching the reference scenario tool, *not* strict FIFA order; fair-play/drawing-of-lots aren't modeled). `scenarioFor()` produces, for a group on its final matchday (≤2 games left), the reference-style grid: for each outcome combination it **enumerates scorelines** (margins 0–6) and unions the finishing positions, so a position cell shows multiple flags when the order genuinely depends on goal margins. Results are memoized in `scnCache` keyed by `groupSig()`. The scenario logic was validated to reproduce the official "Group L scenarios" grid exactly.

### Render vs. dynamic-update split (important when editing UI)
- `render()` rebuilds the whole DOM (sections: Ao vivo / Próximos / Geral / Histórico).
- `refreshDynamic()` updates scores, clocks, and rankings **in place without recreating inputs**.
- `tick()` chooses between them using a signature string (`sig()`): full `render()` only when the set of live/upcoming games changes; otherwise `refreshDynamic()`.
- **`inputFocused()` guard:** never rebuild while a user is typing a prediction — it would lose focus/input. Several handlers (`onblur`, `refreshDynamic`) check this. Preserve this invariant when changing the input grid.

### Match classification (`classify()`)
Each game is `finished` (feed `finished === "TRUE"`), `live` (`time_elapsed !== "notstarted"`), or `upcoming`. This drives which card builder runs: `buildLiveCard` / `buildSoonCard` / `buildHistoryCard`.

### Timezone conversion (`kickParts()`)
The feed's `local_date` is in **each stadium's local timezone**, and WC2026 spans four offsets (Mexico −6; US Central −5; US/Canada Eastern −4; Western −7). `STADIUM_OFFSET` maps `stadium_id` (1–16, from the `/get/stadiums` endpoint) → UTC offset; `kickParts()` converts stadium-local → UTC → Brasília (`BRT_OFFSET` −3). A single fixed offset is wrong (it makes Central/Pacific games an hour or more off). If kickoff times look wrong, check the stadium's offset in the map.

### Teams & flags
`TEAMS` maps the feed's numeric `team_id` → `{ pt: "<name>", code: "<ISO>" }`; `flag()` builds a flagcdn.com URL from the ISO code. Adding/correcting a team means editing the `TEAMS` map.

## Conventions

- All UI strings are Brazilian Portuguese — keep new strings consistent.
- The code style is intentionally terse (minified-ish, multiple statements per line). Match the surrounding density rather than reformatting.
- Treat the Supabase anon key as public (it is, by design) — but do not add service-role keys or other secrets to this client-side file.
