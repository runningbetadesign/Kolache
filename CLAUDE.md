# CLAUDE.md — Kolache

Agent context for Claude Code. Read this before touching anything.

---

## What this is

A casual friend-to-friend wager app. One person posts a bet ("I risk $20 to win $10 that the Lakers win"); friends each take a piece of it at the posted odds. When the outcome is known, everyone confirms it and dough moves.

It's **not** a prediction market and **not** parimutuel. Each acceptance is a tiny 1-on-1 contract between the proposer and that taker, settled at the proposer's posted odds. No pool pricing, no implied probabilities from crowd flow.

Designed to be opened, used, and closed in seconds on a phone. No accounts, no auth, no real money.

---

## Stack

- **One file.** `index.html` is the entire app: HTML, CSS, vanilla JS, no build step, no bundler, no package.json.
- **Firebase Realtime Database** (compat SDK v9.23.0 loaded via `<script>` tag) for shared state across devices, written under `/kolache_v3` using **per-path updates** (no whole-tree blob).
- **localStorage** holds *only* the per-device `currentId` (which player this device is signed in as). All shared state lives in Firebase.
- **Fonts** — Lora (serif) for questions/headings, Outfit (sans) for everything else. Loaded from Google Fonts.
- **No framework, no build, no tests.** Edit, refresh, verify in a browser.

The Firebase config is embedded directly in `index.html` (the project is intentionally open — there's no secret to leak that isn't already client-visible in any Firebase web app).

---

## File map

```
index.html        Entire app. Three logical sections separated by /* ── BLOCK ── */ banners:
                    <style>   theming, layout, components
                    <body>    header, tab bar, three view containers, FAB, sheet, toast
                    <script>  state + render + actions + Firebase sync
```

If you find yourself wanting to add a second file, push back — the constraint that everything lives in one file is intentional. It keeps the project shippable from any static host and trivial to read end-to-end.

---

## Domain model

A **wager** is the unit of content: one person proposing terms that others can take.

In-app state and the RTDB shape both use objects keyed by id (not arrays). Each value carries its key as an `id` field for convenience.

```js
S = {
  players: { [playerId]: Player },
  wagers:  { [wagerId]: Wager },
  currentId: string | null,   // local-only: who this device is signed in as
}

Player = {
  id, name, balance,          // balance is in "dough"; everyone starts at DOUGH_START (1000)
  wins, losses, joinedAt,
}

Wager = {
  id, question, category, closeDate, createdAt,
  proposerId,
  proposerSide: 'YES' | 'NO',
  proposerStake: number,      // how much the proposer puts up (escrowed at creation)
  takerPool:     number,      // total amount takers can collectively put in
                              //   → odds = proposerStake : takerPool
                              //   → equal = even money; risk > pool = laying favorite;
                              //     risk < pool = taking underdog
  acceptances:  { [acceptanceId]: Acceptance },
  status: 'open' | 'pending' | 'disputed' | 'resolved',
  proposedOutcome: 'YES' | 'NO' | null,
  proposedBy: playerId | null,
  confirmedBy: { [playerId]: true },   // proposer auto-confirms when proposing
  disputedBy:  { [playerId]: true } | null,
  resolution: 'YES' | 'NO' | null,
  resolvedAt: number | null,
}

Acceptance = {
  id, takerId,
  takerStake: number,         // how much THIS taker put in (debited at acceptance time)
  acceptedAt: number,
}
```

RTDB layout:
```
/kolache_v3/players/{playerId}                          Player
/kolache_v3/wagers/{wagerId}                            Wager (minus acceptances/confirmedBy/disputedBy)
/kolache_v3/wagers/{wagerId}/acceptances/{acceptanceId} Acceptance
/kolache_v3/wagers/{wagerId}/confirmedBy/{playerId}     true
/kolache_v3/wagers/{wagerId}/disputedBy/{playerId}      true
```

Categories are a fixed list: `Sports, Entertainment, Pop Culture, Finance, Science, Tech, Other`.

**Schema history.** `/kolache` was the original parimutuel `markets[]` shape. `/kolache_v2` reworked it as `wagers[]` but kept the whole-tree-blob write pattern that lost concurrent updates. `/kolache_v3` is the current per-path shape.

---

## Matched-book odds

The proposer specifies two numbers: **I risk $A** and **to win $B**.

- $A is debited from the proposer's balance at creation and held as escrow.
- $B is the total amount takers can collectively put up.
- Anyone can take any portion of $B (limited only by their own balance) at the same A:B odds.

When a taker stakes $t (where `t ≤ remaining = B − Σ takerStake`), the proposer's matched exposure for that acceptance is:

```
propExposure = round(t × A / B)
```

The proposer's *unmatched* escrow (the portion of $A that no taker ever matched) is refunded when the wager settles.

Examples:
- "Risk 20 to win 20" → even money. One taker stakes $20 → proposer's $20 fully matched. Two takers of $10 each → each carries $10 of proposer's exposure.
- "Risk 40 to win 20" → 2:1 favorite. A taker staking $10 puts proposer on the hook for $20.
- "Risk 10 to win 20" → 1:2 underdog. A taker staking $20 puts proposer on the hook for $10.

---

## Lifecycle

```
proposer posts          → status: 'open'        (proposerStake debited)
taker accepts a piece   → status: 'open'        (takerStake debited; multiple acceptances allowed
                                                 until takerPool is fully matched)
proposer cancels (only allowed if 0 acceptances) → wager removed, proposerStake refunded

any party proposes outcome (only after closeDate) → status: 'pending'
                                                    proposer of outcome auto-confirms
other parties Confirm   → added to confirmedBy
other parties Dispute   → added to disputedBy, status: 'disputed'

when every party (proposer + every unique taker) is in confirmedBy
   AND disputedBy is empty                       → settle()
disputed wagers can be re-proposed (overwriting proposedOutcome) and the cycle restarts
```

`settle()` does, for each acceptance:
- proposer's side won → proposer.balance += propExp + takerStake; proposer.wins++, taker.losses++
- proposer's side lost → taker.balance += takerStake + propExp; taker.wins++, proposer.losses++

Plus, refunds any unmatched portion of proposer's stake.

---

## Sync model

`initSync()` attaches two separate listeners — `/kolache_v3/players` and `/kolache_v3/wagers` — and re-renders when either changes. There is **no `saveState()`** function and **no whole-tree write**: every mutation writes only the paths it touches, using `push()` for new children and `update({...})` with a `{path: value, ...}` map for atomic multi-location changes.

Why this matters: the previous design `set()` the entire `{players, wagers, ...}` blob on every action. Two devices acting within the same tick would each write their own copy of "the world," and the second write would clobber the first — bets, balances, and confirmations could silently disappear. Per-path writes don't clobber unrelated nodes, so concurrent actions both land.

Specific patterns:
- **New entity** (player, wager, acceptance): `ref(parent).push().key` → `update({...})` placing the new value at the new key and any sibling writes (e.g. balance debit) in the same atomic update.
- **Field change** (status, proposedBy, confirmedBy/{pid}): targeted `update({path: value})`.
- **Map membership** (`confirmedBy`, `disputedBy`): values are objects keyed by playerId with `true` as the value. To "remove from" the map, write `null` at that path.
- **Settlement** runs from inside the wagers listener (`maybeSettleAll`) whenever a snapshot shows a wager that is `pending`, has no disputes, and has every party confirmed. A `transaction()` on `wager/{id}/status` flips `pending → resolved` so only one device's payouts apply across all clients.

Render order in `initSync()` matters: render immediately from empty state (or whatever's in `localStorage` for `currentId`) before waiting on Firebase, so the screen is never blank.

---

## UI shell

- **Header** — Kolache wordmark + player pill (tap to switch / add player).
- **Tab bar** — Bets / Standings / Resolved. The FAB (+) only shows on Bets.
- **Views** — three sibling `<div class="view">` containers; `switchTab()` toggles `.active`.
- **Sheet** — a single bottom sheet (`#sheet` + `#backdrop`) reused for every modal flow (player list, add player, create wager, accept, propose outcome). Content is injected via `openSheet(html)`.
- **Toast** — single `#toast` element, fired via `toast(msg)`.

All rendering is HTML-string templating with `esc()` for user content. There is no virtual DOM and no component framework — `render()` rebuilds the active views from scratch and re-attaches via `innerHTML`.

Inline `onclick="..."` handlers reference top-level functions. Keep new functions at module scope so the markup can call them.

---

## Conventions

- **Theming** is via CSS custom properties on `:root` — `--bg`, `--surface`, `--text-1/2/3`, `--accent`, `--yes`, `--no`, `--dough`, plus YES/NO light/border variants, plus `--pending`/`--dispute`/`--warn` status colors. Never hardcode hex; reuse the tokens.
- **YES is green (`--yes`), NO is reddish-brown (`--no`).** Keep that mapping anywhere new YES/NO UI shows up.
- **Money** is always rendered via `fmt(n)` → `"1,234 dough"`. Don't print raw numbers.
- **HTML escaping** — every interpolated player/wager string runs through `esc()`. New templates must do the same.
- **IDs** are RTDB push keys (`ref(parent).push().key`) — globally unique strings, no shared counter needed.
- **Dates** — `closeDate` is a `YYYY-MM-DD` string from a date input; timestamps (`createdAt`, `resolvedAt`, `a.acceptedAt`, `joinedAt`) use `firebase.database.ServerValue.TIMESTAMP` via the `TS()` helper (resolves to server-side ms on write).
- **Balance debits are eager.** Proposer debit on create. Taker debit on accept. Settlement only adds back — it never reaches into anyone's balance to subtract.

---

## Workflow

1. Edit `index.html`.
2. Open it in a browser (`open index.html` on macOS, or drag into any browser).
3. Open a second tab / private window as another player to test sync + accept flow.
4. To wipe state: clear the `/kolache_v3` node in Firebase + clear `localStorage`.

Don't introduce: a build step, a bundler, a framework, additional dependencies, a test harness, or split files. If you genuinely need one of those, raise it before doing the work.

---

## Non-goals

- Real money / payments / KYC
- Parimutuel pools or crowd-priced odds (the previous shape — deliberately removed)
- Push notifications / native shells
- Server-side logic — Firebase Realtime DB is the entire backend
- A build pipeline of any kind
- Multi-room / multi-group support (one shared `/kolache_v3` node, everyone is in the same room)
- Comment threads, reactions, social features beyond standings
- Dispute-resolution voting / arbitration (disputes just flag the wager; humans resolve out-of-band)

## Auth and access

The app is a private friend-group game. Strangers are kept out by a Google-sign-in + admin-approval flow.

- **Google sign-in** via Firebase Auth. There is no name-picker — your Google account *is* your identity. The player record under `/kolache_v3/players/{playerId}` is keyed by your `auth.uid` and auto-created on first allowed sign-in using your Google `displayName`.
- **Allowlist** at `/kolache_v3/allowlist/{emailKey}` — emails (lowercased, dots → commas) mapped to `true`. Only signed-in users whose key is on the list (plus the admin) can read or write game state.
- **Self-serve request flow**: a friend signs in with Google; the app checks `/kolache_v3/allowlist/{ek}`; if missing, writes `/kolache_v3/pending/{ek} = {name, email, requestedAt}` and shows a "waiting for approval" screen. Their app listens on `/kolache_v3/allowlist/{ek}` and transitions to the game automatically when the admin flips it to `true`.
- **Admin tab** visible only when `auth.token.email === ADMIN_EMAIL` (`patrick.hermiller@gmail.com`). Shows pending requests with Approve (writes `allowlist/{ek}=true` + deletes `pending/{ek}` atomically) and Deny (deletes the pending entry).
- **RTDB security rules** in `database.rules.json` enforce the allowlist server-side. A user can:
  - Read/write their own `/pending/{ek}` entry (request access, retract).
  - Read their own `/allowlist/{ek}` entry (so the pending → approved listener works).
  - Read/write `/players` and `/wagers` only if their email is in `/allowlist`.
  - Admin can read everything under `kolache_v3/`, write the allowlist, and manage any pending entry.

**Deploying rules.** The GitHub Action only deploys `hosting` (via `FirebaseExtended/action-hosting-deploy`). After editing `database.rules.json`, deploy rules manually from a workstation with the Firebase CLI:

```
firebase deploy --only database --project kolache-e58fa
```

**One-time Firebase Console setup** (not in this repo, has to be done by hand):
- Authentication → Sign-in method → enable **Google**.
- Authentication → Settings → Authorized domains: add `kolache-e58fa.web.app` and `kolache-e58fa.firebaseapp.com` (usually already present); add `localhost` for local testing.

**To bootstrap yourself as the first allowed user**: sign in with Google. If your email matches `ADMIN_EMAIL`, you're allowed automatically (the admin-email check short-circuits the allowlist lookup); your admin tab then lets you approve everyone else.
