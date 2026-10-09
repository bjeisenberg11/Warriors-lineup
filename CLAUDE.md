# CLAUDE.md

## What this is

A lineup app for the **Deerfield 9U Gray Warriors** youth baseball team. Coaches use it to plan who plays which position each inning, track bench time and fairness across the season, set batting orders, and share notes. It works on phones and on an iPad on the fence during games.

The whole app is **one file, `index.html`** (~3,300 lines of inline HTML, CSS and vanilla JS). There's no build step, framework, package.json or server code. The other files are static assets: icons (`favicon.png`, `apple-touch-icon.png`, `icon-192.png`, `icon-512.png`) and `site.webmanifest`.

### Main features (tabs)
- **Lineup**: the inning × position grid (P, C, 1B, 2B, 3B, SS, LF, CF, RF plus bench rows). It runs checks on the card: SHORT (fewer than 9 players), OUT (an absent player is still on the card), duplicates, HOLE (empty position), unplaced players, and fairness warnings. The tab badge counts integrity errors.
- **Who's In**: mark players absent for a game.
- **Batting**: the batting order. It can be copied forward from an earlier game (`mergeOrder`).
- **Trends**: season totals for infield, outfield and bench per player, from the plan or from what actually happened.
- **Notes**: a shared thread between coaches. It shows an unread badge.
- **Game**: date, opponent, home/away, innings, sync status and log, update check, CSV export, archive and restore.

### Domain concepts worth knowing
- **Position assignments**: each regular player has 2–3 allowed spots, primary first (`p.pos`). LF, CF and RF collapse to a single `"OF"` assignment. Anyone can play P and the bench (`eligible`, `openSlot`).
- **Two cards per game**: `slots` is the pregame plan. `actual` is the game-day card, forked from the plan on the first game-day edit. Stats read `actual` if it exists and `slots` otherwise (`effCard`), so a game is never counted twice.
- **Split innings**: a slot holds either a player id (a whole inning) or a 3-element array for thirds of an inning. `fmtInn` uses MLB notation (`2.1` = 2⅓).
- **Call-ups**: guest players with `cu: <gameId>`. They're on the roster for that one game only and are excluded from season stats and fairness.
- **Archive vs. delete**: "delete" archives the game (`arch: 1`). Permanent deletion only happens from the archive, because Firestore keeps no history.
- **Display mode** (`#display` hash): a big read-only view for a dugout or fence iPad. It uses the Wake Lock API to keep the screen on.
- **Print sheet**: the `#print` div, rendered for `window.print()`.
- **Identity**: there are no accounts. A device picks a coach name (stored in `localStorage`, list synced as `coaches` on the team doc), and that name is used as the author of notes.

### Code layout inside `index.html`
- `FIREBASE_CONFIG`, `TEAM_ID`, `BUILD`, `SCHEMA` are near the top of the `<script>`.
- Pure logic lives between `/* ==== LOGIC:BEGIN ==== */` and `/* ==== LOGIC:END ==== */`. That block must stay **DOM-free**: a test harness (`warriors-tests.js`, which is not in this repo) extracts it verbatim.
- After that come the local cache, sync status, mutations, views (`viewLineup`, `viewWho`, …), event handlers, Firestore, and boot (`init()` at the bottom).
- Comments record decisions that were tested on real devices (for example, why PWA standalone mode is **off**: printing doesn't work from an iOS home-screen app). Read them before undoing anything they describe.

## Hosting

- Served as static files from **GitHub Pages** off `main` in `bjeisenberg11/warriors-lineup`. Deploying means committing `index.html` and the assets to `main`; the history shows files uploaded through the GitHub web UI.
- GitHub Pages sends a 10-minute cache header, so the page includes `no-cache` meta tags. The in-app update check (`checkUpdate`) fetches `index.html?v=<timestamp>` and compares its `BUILD` constant with the running one. `reloadLatest()` reloads with a cache-busting query string.
- **Bump `BUILD`** (format `YYYY-MM-DD.HHMM`) on every change you ship. It's shown in the footer stamp, and the update check relies on it.
- The Firebase JS SDK v10.12.0 is loaded at runtime as ES modules from `gstatic.com` (dynamic `import()`), so there's nothing to install.

## Firebase / Firestore sync

**Project:** `warriors-lineup` (config in `FIREBASE_CONFIG`). There's **no Firebase Auth**; access is controlled only by Firestore security rules, which live in the Firebase console, not in this repo. The hard-to-guess `TEAM_ID` (`warriors-k7m2q9x4`) namespaces the data. If `FIREBASE_CONFIG` is set to null, the app runs local-only.

### Data shape
```
teams/{TEAM_ID}                  { schema, roster: [...], coaches: [...], at }
teams/{TEAM_ID}/games/{gameId}   { date, opp, ha, innings, out, order, slots: {...},
                                   actual?: {...}, played?, arch?, bslots?, now?, created }
teams/{TEAM_ID}/notes/{noteId}   { who, text, at, game }
```
Slot keys look like `i3_SS` (inning 3, shortstop) or `i2_BN1` (bench). Rules must cover `teams/{TEAM_ID}/**`, including the subcollections. A `permission-denied` on games logs a hint about this.

### How it works
1. **Boot** (`init`): load the `localStorage` cache (`warriors_lineup_v1`). If the cache is missing or its `schema` doesn't match `SCHEMA`, fall back to the built-in seed roster and the 8/22/26 game. Render right away, then `connect()`.
2. **`connect()`** imports the SDK and opens three `onSnapshot` listeners: the team doc, the `games` collection, and the `notes` collection. Each snapshot replaces the in-memory state `S` and re-renders. Notes are deliberately independent, so a notes failure doesn't block the lineup.
3. **`arrive()`** runs once both the team and games snapshots have arrived, and sets `ready = true`. On the first arrival it **seeds without overwriting**: it uploads the roster only if the server has none, uploads local games only if the server has none, and runs `seedPositionsOnce()` only if no player has assignments yet. It also migrates the legacy single-document `state` JSON format on the team doc.
4. **Writes are field-level.** Every mutation updates local state, calls `commit()` (save to localStorage and re-render), then writes only what changed:
   - `writeGameField(gid, "slots.i3_SS", val)` uses `updateDoc` with a dotted field path, so two coaches editing different cells at the same time both keep their edits. Pass the `DELETE` sentinel to remove a field (mapped to `deleteField()`). If the doc is `not-found`, it falls back to `setDoc` with the whole game.
   - `writeGame(g)` writes the whole game doc (via `copyGame`) and is used when a game is created.
   - `writeRoster()` and `writeCoaches()` use `setDoc(..., {merge:true})` on the team doc. The roster is a single array, so roster edits are last-write-wins.
   - `writeNote`, `removeNote` and `removeGame` set or delete individual docs.
   - Every writer returns early unless `DB && ready`, so nothing is written before the server state is known. That protects server data from a stale cache.
5. **`localStorage` is only a cache** ("mirror of the server, never the source of truth"). The server snapshot always wins. Per-device keys: `warriors_lineup_v1_cur` (selected game), `_me` (coach name), `_seen` (notes read time).
6. **Sync status**: `SYNC` tracks mode (`local` / `connecting` / `live` / `error`), pending writes (`track()`), the last error, and a 12-line log (`note()`), all shown on the Game tab. House rule: never report a bare "offline"; every failure surfaces its error code.

### Safety notes for changes
- Changing `SCHEMA` makes every device ignore its local cache. Server data is unaffected.
- Never add code that overwrites server data on load. Seeding must stay "only if the server is missing it".
- Keep writes at the narrowest field path. Writing whole documents reintroduces the lost-update problem between coaches' phones.
- There are no backups. The season CSV export (`seasonCsv`) is the only way to get data out.

## Working in this repo
- The owner wants changes pushed straight to `main` (no branch or PR, no asking first). Bump `BUILD` with each app change, and always tell the owner the new `BUILD` number after pushing, since that is what Check for update shows. GitHub Pages publishes about a minute after the push.
- If a Pages build gets stuck or fails on GitHub's side, a small real commit to `main` starts a fresh one. Never set Settings → Pages → Branch to None: that unpublishes the site (the app 404s) until a deploy succeeds.
- Don't add a build system or split the file without being asked. Single-file static hosting is intentional.
- To test locally, serve the directory (for example, `python3 -m http.server`) and open `index.html`. It connects to the **live** Firestore project, so edits you make are real team data. To experiment safely, set `FIREBASE_CONFIG = null` or use a different `TEAM_ID`, and change it back before committing.
