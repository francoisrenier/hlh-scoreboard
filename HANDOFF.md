# Game Week Scoreboard: handoff brief

## Goal

Get this board-game-week scoreboard running on a public URL that anyone can open **without any account**. It needs a shared realtime database, so everyone sees the same players, games and totals live.

It was prototyped as a Claude artifact. Artifacts with shared data only work for signed-in members of the owner's Claude organization, which is why it's moving to self-hosting.

## Files you should have next to this brief

- `index.html` is the complete single-file app: HTML, CSS and vanilla JS, with no build step. It uses the **Firebase JS SDK v10 compat** builds from gstatic (`firebase.firestore()`, namespaced API). It already includes the existing scores as an embedded JSON seed (`<script id="seed" type="application/json">`). When the board is empty, it shows an **Import earlier scores** button that writes them with their original document IDs.
- `game-week-backup.json` is a raw backup of the same data (`players`, `games`, each with `id`). It's the source of truth if the seed ever needs regenerating.

## What to do

1. **Firebase project (Frans does this in the console, or via the CLI if logged in)**
   - Create the project, then add a Web app and copy the `firebaseConfig`.
   - Create a Firestore database (region `europe-west1` or similar; the users are in Belgium).
2. **Paste the config** into the `window.FIREBASE_CONFIG = {...}` block near the top of `index.html`.
3. **Firestore rules.** Open read/write, time-boxed to the event:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{db}/documents {
       match /{document=**} {
         allow read, write: if request.time < timestamp.date(2026, 10, 15);
       }
     }
   }
   ```
4. **Host it.** Preferred is Firebase Hosting (`firebase init hosting`, public dir containing `index.html`, no SPA rewrite needed, then `firebase deploy`). Netlify Drop or GitHub Pages are fine alternatives.
5. **Verify** with two browsers or devices:
   - The board is empty and the import button appears.
   - Importing shows 8 players and 15 games.
   - Adding a point on one device shows up live on the other.
6. Optional: add an **Export** button that downloads the current `players` and `games` as JSON, for backups.

## Data model (Firestore)

| Path | Shape |
|---|---|
| `players/{id}` | `{ name, emoji, color, createdAt }` |
| `games/{id}` ranking | `{ name, mode: "rank", ranking: [playerId…], ties: [bool…], winnerOnly: bool, points: [..], createdAt, editedAt? }` |
| `games/{id}` teams | `{ name, mode: "teams", teams: { A: [ids], B: [ids] }, teamWinner: "A"\|"B"\|"draw", ranking: [...A, ...B], points, createdAt }` |
| `games/{id}` quiz | Same as ranking, plus `mode: "quiz"` and `quizScores: { playerId: n }` |
| `quizzes/{id}` | Live quiz: `{ name, players: [ids], status: "live"\|"done"\|"cancelled", startedAt, endedAt? }` |
| `quizzes/{id}/points/{id}` | One document per tap: `{ player, delta: ±1, at }`. Append-only, so simultaneous taps never overwrite each other. |

- `ranking` is the finishing order, winner first.
- `ties[i] === true` means player `i` shares the place of player `i-1`.
- Old games may lack `mode` or `ties`; treat them as `"rank"` with no ties.
- **Totals are always recomputed from `ranking`, `ties`, `winnerOnly` and `teams` with the current rules.** The stored `points` field is informational only, so rule changes apply to past games too.

## Scoring rules (as agreed with Frans)

**Ranking games, by player count:**

| Players | 1st | 2nd | 3rd | Last | Others |
|---|---|---|---|---|---|
| 2 | +1 | – | – | −1 | – |
| 3 | +2 | 0 | – | −1 | – |
| 4 | +4 | +2 | 0 | −2 | 0 |
| 5+ | +5 | +3 | +1 | −2 | 0 |

**Ties:**
- Tied players share a place and each get that place's points (the best place in the group).
- Players tied **for last** all get the last-place penalty.
- If everyone ties, nobody scores.
- The next player after a tie takes their actual position. For example, 5 players with 2nd and 3rd tied scores 5, 3, 3, 0, −2.

**Winner-only option:** the winner gets the 1st-place points above and everyone else gets 0.

**Team vs team:** each winning player gets +1, each losing player −1, and a draw gives 0 to all.

**Quiz:** players collect quiz points live. On finish, they're ranked by quiz score (equal scores become ties) and scored with the ranking table.

A **win** counts for place 1, including a shared first place, or for being on the winning team.

## App features, so nothing regresses

- **Tabs:** Scoreboard (ranking with a score-track token per player, live quiz banner), Log a game (Ranking, Team vs team, or Quiz mode), History (edit and delete each game), Players (name, emoji avatar and colour; a player can only be removed if they haven't played).
- **Editing** a game keeps its original `createdAt` and sets `editedAt`.
- **Confirmations use a custom in-page modal.** Don't switch back to `window.confirm`, which is blocked in some embedded contexts.
- **Listeners** reconnect automatically with backoff if they drop.

## Constraints and preferences

- Keep it a single static file with no framework or build step, unless there's a strong reason.
- It's mobile-first, used on phones around a table.
- Current standings, for sanity-checking the import: Antoine 19, Died 17, Geoff 12, Pierre-Yves 7, Frans 4, Loic 4, Seb 4, Ludo 3.
