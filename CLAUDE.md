# CLAUDE.md — Kolache

Agent context for Claude Code. Read this before touching anything.

---

## What this is

A casual, group-of-friends prediction market. Players bet "dough" (play money) YES/NO on questions they post for each other. Pari-mutuel payouts: winners split the entire pool proportionally to their stake on the winning side.

Designed to be opened, used, and closed in seconds on a phone. No accounts, no auth, no real money.

---

## Stack

- **One file.** `index.html` is the entire app: HTML, CSS, vanilla JS, no build step, no bundler, no package.json.
- **Firebase Realtime Database** (compat SDK v9.23.0 loaded via `<script>` tag) for shared state across devices.
- **localStorage** holds *only* the per-device `currentId` (which player this device is signed in as). All shared state lives in Firebase.
- **Fonts** — Lora (serif) for headings/questions, Outfit (sans) for everything else. Loaded from Google Fonts.
- **No framework, no build, no tests.** Edit, refresh the file, verify in a browser.

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

## Data model

```js
// Shared state (mirrored to Firebase under /kolache)
S = {
  players: Player[],
  markets: Market[],
  currentId: number | null,   // local-only: who this device is signed in as
  _uid: number,               // monotonic id counter, also synced
}

Player = {
  id, name, balance,          // balance is in "dough"; everyone starts at DOUGH_START (1000)
  wins, losses, joinedAt,
}

Market = {
  id, question, category, closeDate,
  creatorId: number | null,   // null = the built-in seed market (anyone can resolve it)
  status: 'open' | 'resolved',
  resolution: 'YES' | 'NO' | null,
  resolvedAt: number | null,
  payouts: { [playerId]: number },   // populated at resolution
  bets: Bet[],
  createdAt, seed?: boolean,
}

Bet = { id, playerId, side: 'YES' | 'NO', amount, at }
```

Categories are a fixed list: `Sports, Entertainment, Pop Culture, Finance, Science, Tech, Other`.

---

## Pari-mutuel payout

When a market resolves to a side:

```
for each winning bet b:
  payout = floor(b.amount / winPool * totalPool)
  player.balance += payout

# W/L: each player who bet on the winning side gets +1 win.
# Players who only bet on the losing side get +1 loss.
# A player who bet on both sides and won counts as a win only.
```

If nobody bet on the winning side, all bets are refunded. Losing-side dough was already debited when the bet was placed (in `placeBet()`), so resolution only *adds* to balances — never subtracts.

---

## Sync model

The sync logic in `initSync()` / `saveState()` exists because of one specific bug: naive Firebase listeners cause writes to bounce back as "remote updates" and overwrite in-flight local edits.

The fix:

1. Each browser tab generates a `SESSION_ID` (random 8 chars) on load.
2. Every `saveState()` writes `{ players, markets, _uid, _by: SESSION_ID }` to `/kolache`.
3. The `.on('value')` listener checks `d._by === SESSION_ID` and skips its own echoes.
4. Real remote updates (from other devices) overwrite `S.players`, `S.markets`, `S._uid` and re-render.

Don't "simplify" this by removing `_by` / `SESSION_ID` — it will reintroduce the echo bug. Recent commit history (`45ab15a`, `78ee999`, `8af7961`) is a record of multiple failed simpler approaches.

Render order in `initSync()` also matters: render immediately from empty state (or whatever is in `localStorage` for `currentId`) before waiting on Firebase, so the screen is never blank.

---

## UI shell

- **Header** — Kolache wordmark + player pill (tap to switch / add player).
- **Tab bar** — Markets / Standings / Resolved. The FAB (+) only shows on Markets.
- **Views** — three sibling `<div class="view">` containers; `switchTab()` toggles `.active`.
- **Sheet** — a single bottom sheet (`#sheet` + `#backdrop`) reused for every modal flow (player list, add player, create market, bet, resolve). Content is injected via `openSheet(html)`.
- **Toast** — single `#toast` element, fired via `toast(msg)`.

All rendering is HTML-string templating with `esc()` for user content. There is no virtual DOM and no component framework — `render()` rebuilds the active views from scratch and re-attaches via `innerHTML`.

Inline `onclick="..."` handlers reference top-level functions. Keep new functions at module scope so the markup can call them.

---

## Conventions

- **Theming** is via CSS custom properties on `:root` — `--bg`, `--surface`, `--text-1/2/3`, `--accent`, `--yes`, `--no`, `--dough`, plus YES/NO light/border variants. Never hardcode hex; reuse the tokens.
- **YES is green (`--yes`), NO is reddish-brown (`--no`).** Keep that mapping anywhere new YES/NO UI shows up.
- **Money** is always rendered via `fmt(n)` → `"1,234 dough"`. Don't print raw numbers.
- **HTML escaping** — every interpolated player/market string runs through `esc()`. New templates must do the same.
- **IDs** come from the synced `uid()` counter (`S._uid++`), not `Date.now()` — Firebase needs them to be stable across clients.
- **Dates** — `closeDate` is a `YYYY-MM-DD` string from a date input; timestamps (`createdAt`, `resolvedAt`, `bet.at`, `joinedAt`) are `Date.now()` ms.

---

## Workflow

1. Edit `index.html`.
2. Open it in a browser (`open index.html` on macOS, or drag into any browser).
3. Open a second tab / private window to test sync.
4. To wipe state: clear the `/kolache` node in Firebase + clear `localStorage`.

Don't introduce: a build step, a bundler, a framework, additional dependencies, a test harness, or split files. If you genuinely need one of those, raise it before doing the work.

---

## Non-goals

- Real money / payments / KYC
- User accounts, auth, OAuth, email
- Push notifications / native shells
- Server-side logic — Firebase Realtime DB is the entire backend
- A build pipeline of any kind
- Multi-room / multi-group support (one shared `/kolache` node, everyone is in the same room)
- Comment threads, reactions, social features beyond standings
