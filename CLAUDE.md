# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file static web app (`index.html`) that displays a public "open games" board for **WCSA — the Middle Tennessee Soccer Alliance**.

**This is a copy of the TSL board (`615jess/TSLOpenGames`), not a fork with its own logic.** The only intended differences are the `<title>`, the header `<h1>`/subtitle, and the `CONFIG` block (its own Apps Script `/exec` URL, and the WCSA contact: 615jess@gmail.com / 615-544-5377). To bring a TSL board change across, copy TSL's `index.html` over this one and re-apply those three spots. The Apps Script behind it runs the **same** code as TSL's but is a separate project bound to a separate Sheet, with its own `WRITE_TOKEN`/`ADMIN_TOKEN` — so a request made here never lands in TSL's queue.

Referees browse uncovered game slots, select one or more, and request them.

There are **two request paths**, and which one is offered depends on the data:

- **On-board submission (preferred).** The board POSTs the request to the Apps Script, which records it and may confirm it outright. Offered **only when every selected row carries a `Game Key`** — the stable per-game id the script matches on. The referee's name and email are then stored in the Sheet, by the assignor, for the games they asked for.
- **Pre-filled email (`mailto:`) or text (`sms:`) — the fallback.** The original path, kept because a listing pushed by an older assignor tool has no `Game Key`, and because the submission can fail. Nothing is persisted server-side; the name/email exist only inside the message the referee sends.

## Architecture

Everything lives in `index.html` — HTML, CSS (in `<style>`), and JS (in `<script>`). There is no build, no dependencies, no package manager, and no tests. Edit the file directly and open it in a browser (or push to deploy) to see changes.

Data flow:
1. On load, `loadGames()` fetches JSON from a Google Apps Script Web App endpoint (`CONFIG.apiUrl`). That script reads the assignor's Google Sheet of open slots and returns an array of row objects.
2. Keys are lowercased/normalized; each row gets a `_id` for selection tracking.
3. `renderBoard()` groups games **by day, then venue** and renders cards. Day first is deliberate: venue-only grouping put Saturday and Sunday in the same block, separated by nothing but the time on each card, and referees kept requesting the wrong day. `gameDate()` builds the date with the local-date constructor, never `Date.parse` of an ISO string, so a Saturday evening game cannot slide into Sunday. Selection state lives in the `selectedIds` Set.
4. `buildEmailBody()` / `buildTextBody()` compose the fallback message; `sendClaim()` / `sendText()` hand off to `mailto:` / `sms:`.
5. `submitClaims()` is the on-board path: it fetches a nonce (`?nonce=1`), then POSTs one `{action:'claim', nonce, name, email, gameKey, role, hp}` per selected game and renders the per-game outcome back into the modal.

### Key conventions

- **`CONFIG` object (top of `<script>`)** is the only thing meant to be edited per-deployment: `apiUrl` (Apps Script endpoint), `assignorEmail`, `assignorPhone`, `boardTitle`.
- **`gf(game, ...keys)`** is the resilient field accessor — it matches sheet column names ignoring case/spaces/`_`/`-`. The Google Sheet's column headers can change wording; add new alias keys to `gf()` calls rather than assuming an exact header. Existing aliases handle things like `start date/time` vs `startdatetime`, `age group` vs `agegroup`, `open roles` vs `roles`.
- **`formatDayTime(dt)`** parses several date formats (US `MM/DD/YYYY`, ISO, time-only, and full JS `Date.toString()` output) and deliberately builds dates with the local-time constructor / string extraction to avoid UTC timezone shifting. Preserve that behavior when touching it — times must display in Central time as entered.
- **Role display**: a sheet value of `CR` renders as "Referee"; anything else (e.g. `AR`) renders as "AR".
- All sheet-derived strings are passed through `esc()` before insertion into HTML.

### On-board submission

- **`claimsSupported(games)` gates the whole feature** on every selected row having a `Game Key`. Never claim against `Match ID`: it is the short game number only when that is unique in the batch, so it can change between pushes and would aim a request at the wrong game. When the gate is false the modal shows only Email/Text, exactly as before — **keep that fallback working**, it is what serves an old listing or an outage.
- **Requests are POSTed one at a time**, not in parallel: the script serialises them on a single lock and a burst trips its own rate limit.
- **`Content-Type: text/plain`** is deliberate — it avoids the CORS preflight Apps Script cannot answer. Do not "fix" it to `application/json`.
- **The nonce is not authentication.** It only stops a script POSTing blind at the `/exec` URL; anyone who loads the board has one. It is dropped after each submission so the next attempt fetches a fresh one.
- **Every outcome is shown, including failure.** A game already filled reports as such and lists nearby same-day alternatives the script returned; a network error says so. Silence would leave a referee believing they have a game they do not have.
- The script decides what auto-confirms (younger-age AR slots) and what waits for the assignor. The board only reports what came back — **do not reimplement that rule here**, it lives in the Apps Script.
- **The instructions banner describes the on-board path, not the fallback.** It also states that a referee's name and email are **recorded with the request** — they are, on the Sheet. The old wording promised the opposite ("not stored or shared online"), which stopped being true the moment requests were recorded; if the storage model changes again, that line changes with it.
- **There is no confirmation email, by design** — one mail per request was too much mail. The modal is therefore the referee's only record, and it must say so. It also reports the game the script **allocated**, taken from the response, not the card that was clicked: several near-identical games can be advertised as one listing, so the game number and even the field that come back may differ from what is on screen.

## Deployment

Hosted from the `615jess/WCSAOpenGames` GitHub repo (GitHub Pages). Deploy by committing to `main` and pushing. The Google Apps Script backend is separate and not in this repo; changing what data appears on the board means editing that script / the underlying Sheet, not this file.
