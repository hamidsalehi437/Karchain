[KARCHAIN-SYSTEM.md](https://github.com/user-attachments/files/31872509/KARCHAIN-SYSTEM.md)
# Karchain
Permissionless Micro-Task Marketplace &amp; B2B Trade Floor Architecture.
# KarChain — System Documentation 

**Permissionless Micro-Task Marketplace · Custom BEP-20 Token Build · Escrow-Backed B2B Trade Floor**

| | |
|---|---|
| **Deliverables** | `index.html` — the worker/poster platform (~4,570 lines) · `admin.html` — the isolated operator console (~3,700 lines) |
| **Stack** | HTML + Tailwind (CDN, with offline fallback) + vanilla JS. No build step, no framework, no backend, no PHP. |
| **Persistence** | Browser `localStorage` under the shared key `karchain.db.v1` — **one local node, two files** |
| **Schema** | v5 (forward-migrates v1/v2/v3/v4 saves automatically, idempotently) |
| **Test status** | **364 / 364 checks green** (e2e 107 · admin 84 · edge 101 · upgrade 72) |
| **Legacy** | `karchain.html` (the frozen v3 deliverable) still runs; its saves migrate into v5 and it never downgrades the node |

---

## 1 · What v5 Adds — the Bazaar Trade Marketplace

v5 overhauls the Bazaar Shop into a **secure, image-supported B2B trade
marketplace with escrow-backed purchases and a physical passcode
handshake** — without touching the decoupled XAMPP/localStorage
synchronisation model (same shared node, same isolated sessions).

1. **Visual asset image core** — `market.items` gain `imageUrl` (nullable).
   The *List Item for Sale* form carries a **Product Image URL / Asset
   Link** input pre-seeded with a clean placeholder tech image URL, so
   empty listings still render beautifully. Cards are reworked into an
   **enterprise product layout**: an ~11-rem high-contrast image frame
   using Tailwind `object-cover`, category + live status chips overlaid,
   name and price on a gradient scrim.
2. **Escrow-backed B2B purchase state machine (§6F `Market`)** —
   *Purchase Item* runs a live balance check in the listed currency
   (KRC included), **debits the buyer only** and moves the tokens into
   their `escrow.Bazaar` tracking array; the merchant is **never paid
   directly**. Listing status flows `AVAILABLE → PENDING_DELIVERY → SOLD`.
   A unique random **4-digit passcode** is minted per transaction
   (uniqueness enforced across all live pools) and is visible **only on
   the buyer's management panel**.
3. **Secure passcode handshake handoff** — the seller's active-sales card
   carries an input module for the buyer's 4-digit code (the physical
   dropoff moment). A correct code releases instantly: **98% of the
   locked pool to the merchant** (a `MARKET_SALE` row on the master
   double-entry ledger), **2% network-marketing commission to the
   merchant's referrer** (`REFERRAL_COMMISSION` row) **plus +50 XP**.
   Wrong codes freeze the pool and leak nothing. A merchant with no
   referrer takes 100% (matching the mission-referral pattern).
4. **Admin control-deck governance** — the `admin.html` **Ops Deck** gains
   a **Bazaar Escrow Monitor**: every frozen `PENDING_DELIVERY` pool
   (thumbnail, item, buyer → merchant, amount + currency, lock age,
   passcode masked `••••` for buyer privacy) with a **Force-release**
   override to settle frozen merchant escrow on disputes or delivery
   communication breakdowns, plus a settled-sale history with an
   *operator release* chip for admin-forced releases.
5. **Full i18n** — every new input label, hint, status chip and splash
   (`ESCROW LOCKED` / `ESCROW RELEASED`) is translated in **en / ps / fa**
   with the slate-900 skin unchanged.

Everything from v4 is retained: KRC BEP-20 registry, Web3 wallet /
Chain-56 enforcement / live `balanceOf`, 98/2 mission referral splits,
Profile Settings, trust scores, KYC desk, the decoupled console, the
dual-role shell, mission lifecycle, i18n RTL, sync outbox, XP engine,
deposit/payout rails and the demo dataset.

---

## 2 · Two-File Architecture, One Local Node

```
htdocs/
├── index.html   platform — Worker / Poster / Bazaar / Profile (all member features)
└── admin.html   console  — ops deck (+ escrow monitor), KYC desk, users & trust, config, chat
        │
        └── both read/write  localStorage['karchain.db.v1']
```

- **Decoupled sessions.** The platform session lives in `DB.session`;
  the console session lives in `DB.admin.console`. Signing into one never
  signs you into the other; a platform member stays signed in across a
  console round-trip (and across upgrades).
- **Console privileges are data, not code.** `admin.html` gates on
  `user.isAdmin` (seeded operator: **admin / admin1234**).
- **Cross-file flows work end-to-end**: a buyer locks escrow on
  `index.html`; the operator force-releases it from the `admin.html`
  monitor; the split, ledger rows, XP and settled history are all visible
  back on the platform. All covered by the test suite.
- **No cross-file coupling at runtime** — only the storage key is shared.
  Either file can boot a fresh node alone (the engine seed is identical,
  byte-for-byte, apart from the three documented file-specific deltas).

---

## 3 · KRC Token & Web3 Layer (v4, unchanged)

- Registry `CFG.CURRENCIES.KRC`: 18 decimals (displayed at 4), 42-char
  `0x…` contract placeholder, `chainId: 56`, `chainIdHex: '0x38'`.
- `Web3.connect` enforces BNB Smart Chain (switch + 4902 add-chain),
  then `refreshKrc()` runs a raw `eth_call` `balanceOf()`
  (selector `0x70a08231`) decoded with BigInt at 18 decimals.
- KRC is a **first-class bazaar currency**: listings may be priced in
  KRC, purchases lock KRC escrow, and referral commissions settle in KRC.
  The simulated deposit/payout rails deliberately exclude KRC
  (chain-native); `import-krc` reconciles the platform ledger to the
  on-chain balance.

---

## 4 · Referral Engine (§6C) — missions **and** bazaar sales

```
mission cleared  ── Referrals.onMissionCleared(job)
bazaar sale      ── Referrals.onSaleCleared(item)          // v5
   └─ both: referrer XP +50, violet ops event, Sync.track(...), split returned
        ├─ missions: worker 98% → EARNING row · referrer 2% → REFERRAL_COMMISSION row
        └─ bazaar:   merchant 98% → MARKET_SALE row (net of commission)
                     referrer 2% → REFERRAL_COMMISSION row (asset = listing currency)
```

- `commissionForSale(item)` routes to the **merchant's** referrer;
  no referrer (or a split that rounds to zero) ⇒ merchant takes 100%.
- Attaching a referrer guards against unknown handle, self-referral,
  double-attach and referral cycles.
- **v5 hardening**: the mission passcode flow now verifies the 4-digit
  code *before* minting the referral split — a wrong attempt can no
  longer leak the referrer's +50 XP.

---

## 5 · Bazaar Escrow Engine (§6F `Market`)

```
AVAILABLE ──buy──▶ PENDING_DELIVERY ──passcode|admin──▶ SOLD

Market.lock(buyer, item)                    Market.release(item, via)
  ├─ live balance check (listed currency)     ├─ buyer pool removed
  ├─ debit buyer (roundCur)                   ├─ 98% → merchant + MARKET_SALE row
  ├─ push {itemId,currency,amount,lockedAt}   ├─ 2%  → merchant referrer + REFERRAL_COMMISSION
  │    into buyer's escrow.Bazaar array       ├─ +50 XP → referrer (violet event)
  ├─ unique 4-digit passcode minted           ├─ status SOLD, soldTo/soldAt set
  │    (retry vs all live pools)              └─ tx.releasedAt / releasedVia recorded
  ├─ status PENDING_DELIVERY
  └─ MARKET_BUY row on the buyer (escrow-locked memo)
```

- **Buyer panel**: the pending card shows the big mono passcode +
  "show this at dropoff" hint. **Seller panel**: the handshake input
  (wrong code → inline error; correct → `ESCROW RELEASED` splash,
  emerald event, `Sync.track('bazaar.release')`).
- **Purchase guards**: not-available / locked-for-another-buyer /
  own-listing / insufficient-balance (each toasts, listing untouched).
- **Image resilience**: empty `imageUrl` renders the built-in Unsplash
  placeholder; a capture-phase `document error` listener swaps broken
  images to an **offline-safe embedded SVG** (`PRODUCT ASSET`) — no
  inline handlers, works with zero connectivity.
- Every account carries `escrow.Bazaar = []` from day one (seed,
  register, wallet-connect, guest, migration) with defensive healing
  inside `Market.lock`.

---

## 6 · Trust Score (§6D) & KYC Pipeline (v4, unchanged)

Base 50 · +5/completed mission (cap +25) · +10 KYC-verified · −15/dispute
· +1 per 60 account-days (cap +5). Tiers: ≥90 Elite Operator · ≥70
Trusted · ≥40 Standard · else Newcomer. KYC submits from Profile
Settings; the console desk approves/rejects.

---

## 7 · Admin Console (`admin.html`)

| Tab | Contents |
|---|---|
| **Ops Deck** | UNDER_REVIEW approval queue (approve → escrow release + referral split + XP; reject → dispute), live event stream (capped 120), **Bazaar Escrow Monitor (v5)** — frozen pools with masked passcodes + Force-release override + settled history with operator-release chips |
| **KYC Desk** | Pending submissions with member context; processed history |
| **Users & Trust** | Every member: trust badge, XP/rank, balances (incl. KRC), KYC chip, referral performance; live search |
| **System Config** | YAML editor; save bumps the version and publishes the global banner; factory reset (confirm-guarded) |
| **Support Center** | Ticket inbox (5 seeded), threads with simulated worker acks, resolve/reopen |

`bazaar-force-release` is operator-guarded (`adminSignedIn`), settles via
`Market.release(item, 'admin')`, logs an amber event and tracks
`bazaar.release` to the sync outbox. The console is English-only by
design; the engine i18n dictionaries stay shared.

---

## 8 · Persistence & Migrations

`Store` keeps the node in one JSON blob under `karchain.db.v1`.
`migrate()` is **idempotent** and heals, in order: v1→v2 admin module →
v2→v3 bazaar/events/outbox/locale/XP-backfill/region metadata → v3→v4
KRC arrays, referral graph, KYC pipeline, web3 settings, console node,
operator provisioning → **v4→v5**: item `imageUrl` / `status`
(derived from `soldTo`) / `tx`, every user's `escrow.Bazaar` array, and
the schema relabel. Corrupt JSON and unreachable storage fall back to a
memory-only session; **reset demo data** rebuilds the v5 seed
(10 jobs, 6 users, 5 tickets, 7 bazaar items with topical images,
2 KYC submissions). The frozen v3 file never relabels a newer node.

---

## 9 · Demo Dataset

- **Users**: `demo_worker` / `demo_poster` (password `demo1234`),
  `kabul_tech` (approved KYC, 5,000 KRC, ETH wallet), `mandawi_trader`,
  `ahmad_k` (referred by kabul_tech, pending Tazkira), operator `admin`
  (`admin1234`).
- **Bazaar (7 items)**: iPhone 12 · Dell Latitude · Anker power bank
  (1,800 AFN by demo_poster) · JBL headphones (55 USDT by ahmad_k — the
  referral-split demo) · HP printer · Galaxy A14 · pre-sold ThinkPad.
  Each carries a topical product image URL.
- **Walk the v5 flow live**: sign in `demo_worker` → deposit AFN → buy
  the power bank (escrow locks, code appears) → sign out → sign in
  `demo_poster` → verify the code → 1,800 AFN releases. Or buy the JBL
  as `demo_poster` and force-release it from the console monitor:
  ahmad_k +53.90 USDT, kabul_tech +1.10 USDT and +50 XP.

---

## 10 · Verification (364 checks, all green)

| Suite | Checks | Covers |
|---|---|---|
| `e2e` | 107 | boot + v5 registry, auth rails (register/guest/wallet), marketplace + filters, Web3 connect / live balanceOf / import / wrong-network, Profile hub, full mission lifecycle with 98/2 split, **bazaar: image listings (with + without URL), escrow lock, buyer passcode panel, seller handshake (wrong → right), 100% no-referrer release, MARKET_SALE row, sync outbox** |
| `admin` | 84 | console gate, ops approvals, **Bazaar Escrow Monitor: frozen-pool render, masked passcode, force-release with 98/2 referral split (53.90 / 1.10 USDT + 50 XP), operator-release history**, KYC desk + trust math, users search, config + banner cross-file, support center, **cross-file KRC referral mission**, decoupled sessions |
| `edge` | 101 | referral guards + rounding (incl. commission-for-sale), Web3 guards (no wallet / reject / call failure / switch / 4902 add-chain), decodeTokenAmount, XP gates, trust tiers, sync outbox, event cap, i18n RTL + all 17 `market.*` keys × 3 languages, auth guards, deposit/payout, **bazaar escrow guards (own listing, insufficient, pending double-buy, sold, bad image URL, wrong passcode + no XP leak, passcode uniqueness, multi-pool accounting), image fallback latch, NaN sweep incl. Bazaar pools** |
| `upgrade` | 72 | **real v3 save → v5 full chain**, v4 → v5 surgical fixture, idempotence with a live escrow pool preserved, reset, corrupt/dead storage, partial legacy heal, shared-node contract across all three files (the frozen v3 never downgrades the node) |

The harness boots the real HTML in jsdom with a MetaMask-shaped
`window.ethereum` mock and drives the actual delegated event routers.

---

## 11 · Extending

- **Mainnet KRC**: replace `CFG.CURRENCIES.KRC.contract` with the official
  42-character address — every call site reads it from the registry.
- **More assets**: append to `CFG.CURRENCIES`; listings, escrow math,
  splits and UI pick currencies up generically.
- **Custom product imagery**: point `CFG.MARKET_PLACEHOLDER_IMG` at a
  branded asset, or edit `CFG.MARKET_FALLBACK_IMG` (embedded SVG) — both
  are registry-driven.
- **Dispute workflows**: the monitor's force-release already records
  `releasedVia: 'admin'`; a full dispute queue can branch on the same
  `tx` record without schema changes.
