# Alshather Control App — Project Reference
> **Purpose of this file:** Single source of truth for the app's architecture, data model, and feature history. Upload this file at the start of any Claude session to restore full context instantly. Update it whenever a feature is added.
> **Last updated:** 2026-07-16

## Deployment
- **Live URL:** https://magical-halva-e46758.netlify.app/
- **Hosting:** Netlify (drag-and-drop deploy of single HTML file)
- **Database:** Firebase Realtime Database
  - URL: https://flatmate-split-project-default-rtdb.europe-west1.firebasedatabase.app
  - Data path: `/aca.json`
  - Project ID: flatmate-split-project
  - **Security (since 2026-07-15):** Rules locked to `auth != null`; app uses anonymous auth via REST (Identity Toolkit `accounts:signUp` + `securetoken` refresh). Web API Key required, stored in localStorage `aca-fbkey`, embedded in share links.

## Stack
- Single-file vanilla JS app (`flatmate-split.html`, ~84KB), no build step, no framework
- Firebase REST API via `fetch` (no SDK); live sync via `EventSource` on the `.json` endpoint
- LocalStorage keys: `aca-v3` (data), `aca-fb` (DB URL), `aca-fbkey` (API key), `aca-fbrt` (auth refresh token), `aca-admin` (admin flag)
- Dark mode via `prefers-color-scheme`; CSS custom properties for theming

## Members (16, constant N=16)
m1 Abdulrahman (treasurer) · m2 Mohammed · m3 Abdulaziz · m4 Turki · m5 Haitham · m6 Bader · m7 Meshari · m8 Sultan · m9 Meshal · m10 Saleh · m11 Waleed · m12 Faisal · m13 Amir · m14 Nawaf · m15 Fahad · m16 Saad

## Gathering schedule (weekly Tuesday rotation, 8 pairs)
Abdulaziz & Turki · Mohammed & Abdulrahman · Haitham & Bader · Meshari & Sultan · Meshal & Saleh · Waleed & Faisal · Amir & Nawaf · Fahad & Saad
Anchor: 2026-06-23 = group index 3. Skips stored per-date; rotation shifts on skip.

## Data model (state shape)
```javascript
{
  members: [{id:'m1'..'m16', name}],
  treasurer: 'm1',
  adminHash: null | sha256(pw+':aca-salt-2025'),
  rentPeriods: [{id, amount6m, startMonth:'YYYY-MM', note}],   // 6-month periods
  expenses: [{id, type:'monthly'|'oneoff', name, amount, paidBy:memberId|null,
              paidMembers:[], poolSettled:bool, note, month|date}],
  settlements: [{id, type:'pool_receive'|'pool_pay'|'member_owes'|undefined,
                 memberId?, from?, to?, amount, date, note}],
  rentPayments: {'YYYY-MM': {memberId: {status:'settled'|'paid', settlementId:string|null}}},
  qattahs: [{id, name, month, amtPerPerson,
             shares:[{memberId, amount, settled:bool, settledBy:'expense'|'cash'|null,
                      cashSettlementId}], note}],
  gathering: {groups:[8 pair strings], anchorDate:'2026-06-23', anchorGroup:3, skipped:[dates]}
}
```

## Balance model (calcBal) — THE core logic
- **Fronted expenses:** payer gets `+unpaidCount × share` credit
- **Qattah:** ALWAYS deducts −amount per share (settled or not)
- **Rent (active period, months up to now):**
  - no entry → deduct −monthlyShare (unpaid)
  - `{status:'settled'}` → deduct −monthlyShare (label only, obligation acknowledged)
  - `{status:'paid', settlementId:xxx}` → deduct; auto-created pool_receive (+) offsets → net 0
  - `{status:'paid', settlementId:null}` → NO deduct (legacy migrated data from old array format)
- **Settlements:** pool_receive +amount · pool_pay −amount · member_owes −amount · legacy from/to transfers
- monthlyShare = amount6m / 16 / 6

## Settled vs Cash-paid pattern (Qattah AND Rent — identical UX)
Modal per round/month: one table, columns Member | Settled | Cash paid.
- **Settle** toggle: marks obligation acknowledged; −amount stays in balance
- **Cash** button: marks paid + auto-creates `pool_receive` settlement (+amount); stores `settlementId`/`cashSettlementId` on the share/entry for undo
- Both reversible; undo deletes the auto settlement

## Qattah pool (qPool)
collected = sum of shares with `settledBy==='cash'` · spent = sum of ALL expenses · bal = col − spent
Pool alert when bal < 0: suggests new Qattah of ceil(short/16/50)*50 per person.

## Feature log
- **2026-06-28 (session 1):** Base app: dashboard, members, rent periods, expenses (monthly/one-off), Qattah rounds, balances with breakdown, admin password, Firebase sync, share links, import/export, gathering card, pool/rent alerts, treasurer.
- **2026-07-01 (session 2):**
  - Expense "Paid by Pool" / "Member paid" toggle; Pool Paid / Pool Settled pills; "Pool settles remaining" button
  - Qattah settled/paid two-button model (replaces single paid checkbox)
  - Balances: "Member owes us" button → `member_owes` settlement (e.g. Nawaf owes 600 from before; deducts immediately, shows in breakdown + history in red)
  - Rent converted to same settled/paid model; rentPayments migrated array→object
  - Rent "Collected to date" progress bar; dashboard rent-collected subtitle
  - Dashboard expenses/person fix (no month filter)
  - **Two file-wipe incidents** — recovered from June-28 transcript + session tool outputs
- **2026-07-16 (session 4) — Phase 1 complete:**
  - **PWA**: manifest.webmanifest, sw.js (network-first shell cache, Firebase traffic untouched), icons (192/512/apple-touch), iOS meta tags, SW registration. Deploy is now a FOLDER (index.html + 5 assets), not a single file.
  - **Activity log** (#8): `state.activity` (cap 200), `upd(fn, logMsg)` second param, new "Log" tab with per-device name selector (localStorage `aca-devname`); logged: payments, settle/undo, expense add/edit/delete, settlements, debts, restores, recurring auto-creates, swaps.
  - **Daily backups** (#9): `fbSnapshot()` on load once/day → `/aca-backups/YYYY-MM-DD.json`, keeps last 7; restore UI in Log tab (admin); `restoreBackup()` runs through migrate().
  - **Recurring expenses** (#3): checkbox on monthly expenses (`recurring`, `recurringId`); `rollRecurring()` on init clones latest instance into current month (idempotent); ↻ Monthly pill; edit modal can stop the series.
  - **Debt aging** (#5): `daysSince()`; age shown on member_owes breakdown lines, history rows "(Xd)", debtor row subtitle "oldest debt Xd".
  - **Partial payments** (#1): Cash buttons (Qattah + Rent) prompt for amount (default = remaining). Partial → pool_receive + tracking (`share.partialPaid/partialIds`; rent entry `{status:'partial', paid, settlementIds}`); button shows amber progress "100/300"; completes to full when remainder paid. qPool counts partials of unsettled shares. calcBal unchanged (pool_receive offsets).
  - **Spending chart + Statement** (#4/#7): 6-month CSS bar chart on Dashboard; "Statement" button → monthly summary modal with copy-for-WhatsApp.
  - **Gathering swaps** (#10): `gathering.swaps={date:groupIdx}` override honored by gGroupIdx; Swap button next to Skip.
  - Full runtime smoke tests: 12/12 pass.
- **2026-07-15 (session 3):**
  - Fixed transcript-escaping corruption (`\"` → `"`, `\\u` → `\u`) that made the deployed page blank
  - Fixed broken el() call in renderDashboard
  - Firebase anonymous auth (Option A): fbAuth() with token cache + scheduled refresh + EventSource auth_revoked resubscribe; API key field in setup modal; share links carry {u:url,k:key} JSON (legacy plain-URL links still parse); connection pill in header now clickable → opens setup modal
  - Full runtime smoke tests pass (all 6 tabs + balance scenarios)

## Firebase console setup (completed 2026-07-15)
1. Web app registered → Web API Key obtained
2. Authentication → Sign-in method → Anonymous enabled
3. Rules: `{"rules":{".read":"auth != null",".write":"auth != null"}}`

## Known quirks / gotchas
- `migrate()` converts legacy rentPayments arrays to `{status:'paid', settlementId:null}` = no balance deduction (preserves old balances)
- `init()` merges member names between Firebase and localStorage, preferring whichever has more real (non-"Person N") names; falls back to INIT names
- Old share links (plain btoa(url)) still work but don't carry the API key → users see "Connecting..." until they use a new link
- Python `open('w')` truncates before writing — NEVER write the output file directly; write to /tmp, verify, then copy (cause of both file wipes)

## Deploy structure (since 2026-07-16)
Deploy the FOLDER `alshather-pwa/` to Netlify (drag the whole folder): index.html, manifest.webmanifest, sw.js, icon-192.png, icon-512.png, apple-touch-icon.png.
iOS install: Safari → Share → Add to Home Screen.

## Roadmap ideas
- ~~PWA~~ DONE 2026-07-16
- Push notifications (#6) — Phase 2 headline (needs server function; iOS requires installed PWA, iOS 16.4+)
- Possible future: Capacitor App Store wrapper (needs Apple dev account + native features to pass review 4.2)
- Possible future: multi-group public product (needs accounts, group creation, variable N, per-group rules)
