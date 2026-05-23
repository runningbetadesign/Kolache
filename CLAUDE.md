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
- **Firebase Realtime Database** (compat SDK v9.23.0 loaded via `<script>` tag) for shared state across devices, written under `/kolache_v2`.
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

```js
S = {
  players: Player[],
  wagers: Wager[],
  currentId: number | null,   // local-only: who this device is signed in as
  _uid: number,               // monotonic id counter, also synced
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
  acceptances: Acceptance[],
  status: 'open' | 'pending' | 'disputed' | 'resolved',
  proposedOutcome: 'YES' | 'NO' | null,
  proposedBy: playerId | null,
  confirmedBy: playerId[],    // includes the proposer of the outcome by default
  disputedBy:  playerId[],
  resolution: 'YES' | 'NO' | null,
  resolvedAt: number | null,
}

Acceptance = {
  id, takerId,
  takerStake: number,         // how much THIS taker put in (debited at acceptance time)
  acceptedAt: number,
}
```

Categories are a fixed list: `Sports, Entertainment, Pop Culture, Finance, Science, Tech, Other`.

**Schema note.** Old data under the Firebase path `/kolache` used a parimutuel `markets[]` model and is incompatible. The new app writes to `/kolache_v2` so the two don't collide.

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

The sync logic in `initSync()` / `saveState()` exists because of one specific bug: naive Firebase listeners cause writes to bounce back as "remote updates" and overwrite in-flight local edits.

The fix:

1. Each browser tab generates a `SESSION_ID` (random 8 chars) on load.
2. Every `saveState()` writes `{ players, wagers, _uid, _by: SESSION_ID }` to `/kolache_v2`.
3. The `.on('value')` listener checks `d._by === SESSION_ID` and skips its own echoes.
4. Real remote updates (from other devices) overwrite `S.players`, `S.wagers`, `S._uid` and re-render.

Don't "simplify" this by removing `_by` / `SESSION_ID` — it will reintroduce the echo bug. The Diet-tracking-style amnesia where a write you just made bounces back and clobbers the next write is exactly what this prevents.

Render order in `initSync()` also matters: render immediately from empty state (or whatever is in `localStorage` for `currentId`) before waiting on Firebase, so the screen is never blank.

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
- **IDs** come from the synced `uid()` counter (`S._uid++`), not `Date.now()` — Firebase needs them stable across clients.
- **Dates** — `closeDate` is a `YYYY-MM-DD` string from a date input; timestamps (`createdAt`, `resolvedAt`, `a.acceptedAt`, `joinedAt`) are `Date.now()` ms.
- **Balance debits are eager.** Proposer debit on create. Taker debit on accept. Settlement only adds back — it never reaches into anyone's balance to subtract.

---

## Workflow

1. Edit `index.html`.
2. Open it in a browser (`open index.html` on macOS, or drag into any browser).
3. Open a second tab / private window as another player to test sync + accept flow.
4. To wipe state: clear the `/kolache_v2` node in Firebase + clear `localStorage`.

Don't introduce: a build step, a bundler, a framework, additional dependencies, a test harness, or split files. If you genuinely need one of those, raise it before doing the work.

---

## Non-goals

- Real money / payments / KYC
- Parimutuel pools or crowd-priced odds (the previous shape — deliberately removed)
- User accounts, auth, OAuth, email
- Push notifications / native shells
- Server-side logic — Firebase Realtime DB is the entire backend
- A build pipeline of any kind
- Multi-room / multi-group support (one shared `/kolache_v2` node, everyone is in the same room)
- Comment threads, reactions, social features beyond standings
- Dispute-resolution voting / arbitration (disputes just flag the wager; humans resolve out-of-band)
