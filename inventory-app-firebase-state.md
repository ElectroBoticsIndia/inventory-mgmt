# ElectroBotics Inventory (Firebase) — build state

Single file `inventory-firebase.html`, ~7,600 lines, GitHub Pages under `electroboticsindia`.
Firebase project `electrobotics-student`, shared auth and `users` collection, all data under
`inv_*`. **All eight slices are built**, plus the student link, bill delete, the printable
bill and purchase-order tracking. Verified by 1,487 tests in eighteen Playwright suites driven
against a stubbed Firestore.

**Built since 22 Sep 2026 — each has its own doc:** ElectroBotics Modules with recipes and
auto-consumed parts (`inventory-eb-modules.md`); split quantities in quarters
(`inventory-split-quantities.md`); the Dashboard's Kits in preparation card
(`inventory-kits-in-preparation.md`); vendor choice, module History, the current vendor as a
buying option, kit inflation and kit selling price (`inventory-vendor-history-kit-pricing.md`);
the Firestore read budget (`firestore-read-budget.md`). The Dashboard order is now stat cards
→ Kits in preparation → Module Requirement → Low Stock → charts. The combined rules file has
26 paths (`inv_pos` and `inv_item_log` added).

## Roles — tutor equals admin in THIS app

17 Sep 2026. A tutor has every admin feature in the inventory app: every screen, costs
and margins, add/edit modules, centres, kits, orders and vendor options, Min Qty, Settings,
and every bill including delete. The account menu still shows their real role. **The student
app's admin/tutor split is unchanged**, in its rules and its app.

```js
const TUTOR_FULL_ACCESS = true;
function canSee(){ return role==='admin' || role==='tutor' }        // reads
function canMoveStock(){ return role==='admin' || role==='tutor' }  // qty-only writes
function canEdit(){ return role==='admin' || (TUTOR_FULL_ACCESS && role==='tutor') }
```

Three names are kept because they mark what each gate used to cost under the rules, and
because narrowing one is how the split comes back. `canSee()` and `canMoveStock()` were
always safe to widen in the app alone. `canEdit()` is the one that needed the rules to move.

⚠️ **`canEdit()` and the rules move together.** The rules grant all of this through a single
function, `invManager()`, which every `inv_* ` write reads through — so restoring the split is
one line (`invManager()` → `isAdmin()`) and republish. Narrowing one side without the other
gives either dead buttons or an open door, and the open door is the dangerous direction,
because hiding a button does not stop a hand-crafted request. See `firestore-rules-notes.md`.

The old qty-only field mask on `inv_items` is **gone** — it was what limited a tutor to the
stock map while costs sat on the same document, and it went when both roles became equal here.
Anything that adds a field to `inv_items` (`orderSnoozeUntil` is the most recent) therefore
needs no rule of its own. Reinstating that mask would silently make those fields admin-only
while the app still drew the buttons.

Every admin-only save function is hard-guarded on `canEdit()` inside the function, not merely
hidden.

⚠️ **The rules file is edited in the Firebase console, which this workspace cannot read.**
Pull the live copy before changing it. A stale copy on 22 Sep produced a correct-looking
`inv_pos` block written against a field mask that no longer existed, and a documented
tutor/admin drift that did not exist either.

## Flat navigation (20 Sep 2026)

The Admin drilldown is gone. Seven top-level nav items, in this order — **Dashboard,
Inventory, Kits, Centres, To Order, Sell, Settings**. What used to be Admin's four tabs are
now four of those seven, and `renderAdmin()` was split into `renderDash()`,
`renderInventoryScreen()` and `renderOrderScreen()`. Landing screen is `dash`.

The nav is **driven off one config**, so there is no second list to drift:

```js
const BUILT = { dash:true, inventory:true, kits:true, centres:true, order:true, sell:true, settings:true };
const NAV = [
  {id:'navDash',      screen:'dash',      under:['dash']},
  {id:'navInventory', screen:'inventory', under:['inventory']},
  {id:'navKits',      screen:'kits',      under:['kits','kit']},
  {id:'navCentres',   screen:'centres',   under:['centres','centre']},
  {id:'navOrder',     screen:'order',     under:['order']},
  {id:'navSell',      screen:'sell',      under:['sell','bills']},
  {id:'navSettings',  screen:'settings',  under:['settings']}
];
```

`under` is what keeps a drilldown's parent lit: Bills highlights **Sell**, a kit or centre
detail highlights its list. `applyNav()` and `showLogin()` both iterate `NAV` — the old
hard-coded `['navKits','navCentres','navSell','navAdmin']` in `showLogin()` was left behind by
the rename and threw a null `TypeError` **before** `#loginError` was written, so a parent
signing in saw an empty login box instead of a reason. That is the failure mode to watch for
after any nav change.

Three screens stay drilldowns and have no nav button of their own: `kit`, `centre`, `bills`
(`const DRILL = ['kit','centre','bills']`).

**Purchase is called To Order everywhere a user can see it** — the screen title, the nav, the
dashboard stat card, the Kit Orders note, the empty-orders line. The *code* still says
purchase (`purchaseList`, `poMode`, `exportPurchaseAll`, `renderPurchase`) and so do genuine
domain phrases — purchase rate, Receive purchase, Purchase Spend by Supplier, and the vendor
"Purchase order" WhatsApp message, which is a real PO and should keep that name.

**Update Stock is the name of the dialog, everywhere.** Inventory rows, Component Details,
the centre card and both centre-detail row buttons all say it; the old **Stock**, **Receive**
and **Move** labels are gone. Only the two buttons that pass `forceMode:'Transfer'` still say
**Move stock**, because that is the one thing they do.

### The phone header

On a phone the seven buttons **drop to their own full-width row below the logo**, four across
and the last three centred under them, **with their labels visible** — the old
`@media(max-width:600px){.nav-btn span{display:none}}` is gone. The account avatar stays on
the top row. This needed `<nav class="header-nav">` as a sibling of `.header-left` and
`.header-right` (the account menu used to sit in the same box as the nav buttons), then
`flex-wrap` on `.header-inner` with `order:3; flex:1 0 100%` on the strip. Buttons go
icon-over-label at 9px mono, 44px tall. Asserted: seven visible, every label rendered and
unclipped, two rows, ≥44px, no sideways scroll.

## Screens

| Nav | What's there |
|---|---|
| **Dashboard** | Four stat cards, three Chart.js charts, collapsible Low Stock and Module Requirement cards. |
| **Inventory** | Modules (Stock-by-centre and Costs views) and Stock Log. Row actions: icon-only Details and Edit plus a highlighted Update Stock; the row itself opens the three-tab Component Details dialog. |
| **Kits** | Card grid with can-build per kit; kit detail with the component table **and a search across every displayed field including status**; kit form; BOM editor with the searchable picker. Sub-tab: **Kit Orders**, whose **Modules Short badge is a button** — it jumps to To Order with `poMode='kit'` already applied. |
| **Centres** | Card grid (central store first, then active, then inactive); centre detail with three stock views and transfer-or-buy advice; centre form. The stock table names the centre in its own column heading and drops the duplicate Central column when the central store is the centre being viewed; **tapping a row opens Update Stock**. |
| **To Order** | Two tabs. **To order**: vendor-grouped list, three chips, editable quantities, an **On order** column, **Place order / Place & WhatsApp** per vendor, Excel per vendor or all, and a **Parked for later** strip. **Ordered**: the log of every order placed, searchable, Still due / Every order, Excel. See `inventory-purchase-orders.md`. |
| **Sell** | Centre selector, searchable pick list showing stock at that centre with a **− qty + stepper on every row**, **student-search on the customer field** (`inventory-student-link.md`), cart with editable qty and rate, bill saved as one transaction, and a **running bill bar pinned to the bottom on a phone** (`inventory-sell-picker.md`). Three save buttons: **Save Bill / Save & WhatsApp / Save & Download PDF**. **A new bill defaults to Pending.** |
| **Bills** | Reached from Sell. List with All/Paid/Pending chips and pending value; student ID shown on linked bills; **Bill** (printable PNG/PDF — `inventory-printable-bill.md`); bill detail; editable payment status; **Delete bill** (`inventory-bill-delete.md`); Excel export. |
| **Settings** | The tunables plus the student-app payment switch. |

`saveSale` now takes `after` — `'none' | 'wa' | 'pdf'` — not the old boolean. `'pdf'` opens
the printable sheet and downloads it in one step. The three buttons are deliberately three
different colours (teal, WhatsApp green, near-black) so they are not mistaken for each other
on a phone, and there is an assertion on exactly that.

## The Update Stock dialog

One dialog, four modes, and the most-changed part of the app this week. Four write-ups:

**It says what it will do** (`inventory-stock-recount.md`). A recount appeared to *add* to
the stock it should have replaced. The write was never additive; `writeMove`'s recount branch
has always been `qty[b.to]=q`. The dialog reset to "Receive purchase" at Central Store on
every open, so a counted figure could land in Receipt mode (a real add) or at the wrong
location (a right balance in the wrong pile, which reads as an add on the company total). The
dialog now remembers the mode and location you chose for the life of the page, and Save states
its intent — "This will **ADD** 40 to Central Store (250 → 290)" in amber versus "This will
**SET** Central Store to 40 (250 → 40)" in teal — with the button renamed to match.

**Receive purchase carries the costing** (`inventory-receive-costing.md`). Three parts, ruled
off from To / Quantity below. First the **buying history** full width, as a table — one row
per vendor with the purchase rate and EB cost in their own tinted columns, plus ship/GST/
handling, all-in, invoice, date and quantity. Below it **This purchase** beside **Proposed
update**, the same seven fields each. The left is what the batch cost and goes to
`inv_purchases`; the right is what the module costs from now on and goes to `inv_items`. Both
open holding the module's current cost, so doing nothing keeps the cost where it is, and
**← Copy from this purchase** carries the batch across in one click. Choosing a vendor fills
in the SKU last bought under them; clicking a history row fills both sides.

**It can be opened from an order** (`inventory-purchase-orders.md`). Receiving against a
purchase order comes in here prefilled and goes back to the order on close — receiving never
routes around the one dialog that writes a movement row.

**Sell shows what you picked, where you picked it** (`inventory-sell-picker.md`) is the
matching fix on the other screen — steppers on the pick rows and a running bill bar on a
phone.

## The migration card is gone (19 Sep 2026)

Removed at Ganesh's request, along with everything only it used — about **1,030 lines**: the
card's markup and CSS, all eleven importers, `migCheck` / `migRun` / `migAnalyse`, and the
`parseCSV` / `csvObjects` / `commitAll` / `normDate` helpers whose only callers were the
importers. `build/test2.js` (127 assertions) went with them, and three sections of `test.js`.

**Three things were caught in the blast radius and had to be rescued**, which is the reason
to do a deletion like this with a reference check rather than by eye:

- **`uniqList()`** sat inside the migration block but is used by every filter dropdown in the
  app. Moved out, next to the other small helpers.
- **`migBumpCounter()`** was called by `saveItem`, not just by the importers: when an admin
  types `EB-M-050` by hand, the counter has to jump past it or the next auto-generated code
  collides. Kept and renamed **`bumpCounter()`**.
- **`.mig-note`** styled the live "showing the last 90 days · Load everything" line under
  both windowed tables. Kept and renamed **`.win-note`**.

The method that found them: extract the block, list every name it defines, grep each one
against the rest of the file, and do the same in reverse for what it borrows. Worth repeating
for any large removal here.

**If a one-off import is ever needed again**, the last build that still has the importers is
the file delivered on 19 Sep before this change.

## Invariants worth not breaking

**Kits are built at the central store and nowhere else.** `kitStats`, the BOM editor, the
orders table, the requirement card and the To Order list all measure against `centralQty()`.
Stock at a teaching centre is deployed — it cannot build a kit and cannot refill another
centre. Tested with 3 at central and 99 at each centre: the answer is 3.

**Every quantity change goes through a movement row.** No balance is written without one
behind it. A new module seeds an empty `qty` map and its opening stock arrives as a Receipt —
writing both would double-count, which it briefly did. Deleting a bill writes a **reversal**
rather than removing the sale movement, for the same reason: the log has to explain the
balance on its own. Receiving against a purchase order goes through the Update Stock dialog
for exactly this reason — a "mark received" shortcut would create stock out of nothing.

**A recount SETS, every other mode adjusts.** `writeMove`'s final `else` is the recount and
it writes an absolute figure. When a balance looks like it was added to rather than replaced,
the Stock Log says which mode ran and at which location — the answer has never yet been in
the transaction. See `inventory-stock-recount.md`.

**What a batch cost and what the module costs are two different numbers.** A receipt records
the first in `inv_purchases` and the second in `inv_items`, from two separate sets of fields,
because one invoice must not silently reposition every selling price derived from the module.
A test saves a receipt with deliberately different rates on the two sides, so a crossed wire
cannot pass.

**A hand-set EB cost is never overwritten by a cost change.** `saveReceiptCosting` writes a
new `ebCost` only when `priceMode !== 'manual'`; the same rule holds everywhere a cost moves.

**Location keys in the `qty` / `minQty` maps are always `docId()`-normalised.** This is the
bug that made `EB-CHN-002` show 0 units after real transfers: the code went in raw from the
dropdown (`EB-CHN-002`) and was read back normalised (`EB-CHN-2`), so the stock existed under
a key nothing looked at. It was invisible for `HQ` and `EB-MPK` because `docId()` is a no-op
on codes with no digits. Fixed at every layer — `normMap()` folds legacy raw keys on read (and
drops them the first time the module is written again), all transactions canonicalise, every
write site calls `docId()`, every `*FromDoc` sets `_id: docId(d.id)`, and `selCode` /
`setSelCode` canonicalise, which is the one place a display code and a document id meet.

**Every transaction validates on a fresh read.** `writeMove`, `saveSale` and `deleteBill`
re-read the module inside the transaction and work from what Firestore holds right then, not
the page's cache. That is what replaces Apps Script's `LockService`, and it is stronger: two
people billing at once cannot collide on a bill number, and a tab open all afternoon cannot
oversell. Tested by setting the local cache to a false 99/999 and asserting refusal.

**Anything that is not the stock itself is written outside that transaction, always.** The
cross-app payment write, the receipt's cost update and buying-history row, and the purchase
order's `received` count. A failure there must never roll back — or hide — stock that is
physically on the shelf, so each reports inline instead: *"Stock received ✓ — but the buying
history row could not be written: …"*.

**The To Order chips are derived from the list they summarise.** All three count through
`purchaseList(mode)` — the same function the table renders from — so a chip number is the row
count by construction. The Sheets app computed them separately and they disagreed.
`suggested = max(forKits + forCentres − central − onOrder, 0)`; stock and anything already on
order are each subtracted exactly once.

**An order's state is computed, never stored.** `poState()` derives cancelled / received /
partial / open from the lines, so an order cannot sit in a state its lines contradict. A
status field would go stale the first time a receipt was booked.

**Zero is absence.** A location counted to zero loses its map key; a cleared Min Qty is
deleted rather than stored as 0; a blank Round Off Rate stays `null`. Storing any of these as
0 silently changes what they mean.

## Conventions in force

`esc()` escapes quotes as well as `& < >`; `jsq()` for every inline-handler argument.
Doc ids normalise digit runs (`EB-M-001` → `EB-M-1`) with the padded code kept for display.
Sequence numbers come from `inv_meta/counters` inside a transaction.
`inv_moves` and `inv_sales` load a 90-day window with a "Load everything" button, because
both grow without bound and the project is on Spark.
Restricted actions are hard-guarded inside the function, not just hidden in the UI.

**One WhatsApp composer for the whole app.** `#waModal` is shared by Bills, To Order, the
stock top-up list and the kit shortfall, so a change to it lands everywhere at once — which
is what made moving Copy to the dialog head a single edit. Copy is an `.icon-btn` in
`.modal-head-actions`, left of the close button; having no label, it reports success by
flashing green and changing its tooltip, and resets on reopen.

**Closing a child dialog returns to its parent** (`inventory-dialog-chain.md`). `modalBack`
holds one return function per dialog and `openChildModal(parent, child, open, reopen)` sets it
up, so Edit → Update Stock → close lands back on Edit rather than dismissing the whole chain.
Any dialog that can be opened both on its own and from a parent must `delete
modalBack['<id>']` at the top of its open function, or it inherits a stale parent.

**A save flashes the row it changed** (`inventory-row-flash-kit-search.md`). `flashRow(code)`
finds every `tr[data-code]` for that module and restarts a CSS animation on it with
`void el.offsetWidth`. CSS animations outrank normal declarations in the cascade, which is why
the flash wins over the row's own background.

**A dialog's decisive sentence belongs outside `.modal-body`.** That element is
`max-height:64vh; overflow-y:auto`, so anything in it can sit below the fold on a phone while
the footer stays pinned — which is exactly the text the user would have needed to read before
pressing the button. The Update Stock confirmation lives between the body and `.modal-msg`,
with a test asserting it is above the Save button at 375px.

**A `position:fixed` bar must be taken down when its screen is left.** The Sell bar is the
only one, and `go()` strips its `.show` class for any screen but `sell`. Anything similar
added later needs the same line, or it floats over the rest of the app.

⚠️ **Class-name prefixes are shared across a 6,500-line file.** `.rcpt-` belongs to the
printable bill sheet, which has a test asserting none of its rules use a CSS variable
(html2canvas resolves `var()` inconsistently). The Receive panel first used `.rcpt-` and
broke that test from the other end of the file; it is `.recv-` now. Grep the prefix before
claiming one.

**CSS layout traps, all now asserted.** `.form-row label` is uppercase mono and `.form-row
input` is `width:100%`, so a checkbox row needs `.form-row label.set-check` /
`.form-row .set-check input[type="checkbox"]` to win. `.filter-group` is a *column* flex
container, so a `flex: 1 1 130px` on a select inside it becomes a **height** — that is what
made the Sell category filter 130px tall. `.form-row select` sets an explicit white
background, which overrode the browser's disabled styling until `select:disabled` was added.
The sticky-header rule repaints every `thead th` with `--surface-alt`, beating
`th.price-cell`'s accent styling — which left **EB COST white on pale grey and unreadable**;
fixed with `.table-wrap.vscroll .data-table thead th.price-cell`, and the test *measures the
contrast ratio* (≥4.5) rather than trusting the stylesheet. **A grid item's default
`min-width` is `auto`**, so a `white-space:nowrap` descendant sets a floor the track cannot go
below — a nowrap line in the Sell pick row pushed the card to 400px on a 375px screen until
`.sell-grid>*{min-width:0}` was added. The symptom there is the whole page scrolling sideways,
a long way from the cause. And **`@media(max-width:600px){.table-wrap .data-table{min-width:600px}}`
is global**: a table restyled to stack into cards on a phone must opt out of it by name, or
the cards stay 600px wide and every value sits off-screen while the labels look correct.

**A filter dropdown is only as good as the call that fills it.** The centre-detail Category
and Supplier selects were empty for a week because `renderCentreDetail()` never called
`buildCombos()`. Any screen that renders a filter needs the populate call in its own render
path — inheriting it from whichever screen happened to run first is not a plan.

It happened again on 26 Sep: the Dashboard's **Low Stock → Location** list was empty,
because the Dashboard is the screen the app opens on and nothing on that path called
`buildCombos()` — the list only filled once Inventory had been visited or something saved.
`renderLow()` now fills it itself (`fillSelect` keeps the current choice, so it is free to do
on every draw) and it was taken out of `buildCombos()` so it has one owner. An audit of every
list `buildCombos()` fills found no other gap: centre detail, Inventory, the module form,
Update Stock and the kit order form each fill their own. `test18` asserts the list is full on
first landing with no other screen visited — the real path, and the one earlier tests missed
because their setup happened to visit another screen first.

## Architecture note

The whole app is a **classic script**; a single `<script type="module">` at the end imports
the Firebase SDK, sets `window.FB` and calls `bootApp()`. Module scope is not global, so going
fully modular would have meant rewriting every inline `onclick`. Verification is therefore two
checks: `vm.Script` on the classic block, `node --check` on a `.mjs` copy of the module block.

## Test suites

`build/verify.sh` runs all of it: parse checks → rebuild the stub harness → test.js (72,
shell/roles/flat nav/phone header/EB COST contrast) → test3.js (155, modules & stock) →
test4.js (133, kits/To Order/sell/dashboard) → test5.js (79, UI refinements + the
location-key regression) → test6.js (118, student link, bill delete, the Pending default) →
test7.js (86, row actions, Details dialog, Stock Log coverage) → test8.js (109, the printable
bill, the WhatsApp composer's head, and that the migration card and its helpers are gone) →
test9.js (20, real rasterisation) → test10.js (47, recount overwrites, the remembered mode and
location, the confirmation wording) → test11.js (54, the Sell pick-row steppers and the
running bill bar) → test12.js (115, the Update Stock rename, the Receive history table, the
two costing columns and where each one is written) → test13.js (38, the dialog return chain)
→ test14.js (56, the row flash, the Kit Components search, the centre column rename and the
centre filters) → test15.js (90, purchase orders end to end). **1,172 assertions, all green
at 22 Sep 2026.**

A full `verify.sh` run takes several minutes end to end — give it a generous timeout rather
than assuming it hung.

**The role assertions encode the current policy, and they have been inverted twice.** If one
starts failing, the app and the rules have drifted apart — check which one moved.

**The stub is not Firestore.** Gaps found so far and since filled: `addDoc` was a no-op,
transactions had no `delete`. When a new Firestore call is used for the first time, check the
stub implements it before trusting a green run — and always run `verify.sh` rather than a
single suite, because it is what rebuilds `build/test.html` from the current HTML. Editing the
HTML and re-running a screenshot script without that rebuild is the quiet way to spend a round
debugging a fix that was never loaded.
`window.__THROW_COLL = '<collection>'` makes writes to one collection fail, which is how the
receipt's failure path is tested.

**Playwright auto-dismisses browser dialogs.** `onSellCentreChange()` calls `confirm()` when
the bill is not empty, so a test that switches centre mid-bill must
`page.once('dialog', d => d.accept())` or the switch silently reverts and the assertion looks
like an app bug. The same applies to Close short and Cancel order.

**Assertions on table headings must lowercase both sides.** `th` text is uppercased in CSS, so
comparing against `'Central'` passes or fails for the wrong reason — one check here did pass
for the wrong reason before it was caught.

**`test9.js` serves html2canvas and jsPDF off disk** (installed as dev dependencies) so the
bill really rasterises. ⚠️ Playwright checks routes most-recently-registered **first**, so
the catch-all abort must be registered *before* the specific fulfils, or everything aborts.

## Still open

- **The Firestore rules have never been executed against a real engine.** The emulator could
  not be downloaded in the build sandbox (`storage.googleapis.com` blocked). The file was
  coverage-checked by hand and the student half diffed line-for-line against production after
  every change, but the logic is unproven. `rules/rules-test.mjs` (54 assertions) runs
  elsewhere; three Rules Playground checks are in `firestore-rules-notes.md`.
- **`inv_pos` is in the combined rules file but not yet published** as at 22 Sep 2026.
  Until it is, placing an order fails with a permission error. 25 paths after that edit.
- The rules still grant `write` on `inv_moves` and `inv_meta` more widely than the app now
  needs — that breadth existed for the importers' re-import path. Narrowing it is safe and
  optional; nothing in the app depends on it any more except `counters`. Note that the same
  uniform breadth is what the `invManager()` invariant rests on, so narrow the *function*,
  not individual blocks.
- Charts do not render in the test sandbox (Chart.js CDN is blocked there); they are guarded
  individually with try/catch so a failure cannot blank the dashboard.
- Deleting a bill does not restore a `componentHistory` archive it had displaced — see
  `inventory-bill-delete.md`.
- Offered for the Sell screen and not taken up: a "Picked (n)" filter chip, two-row cart lines
  on a phone, remembering the selling centre beyond `lastCentre`'s page life.
- Two apps share one Google sign-in and one session, by choice — Ganesh looked at the options
  on 21 Sep and decided to leave it.
- Not built, and deliberately: part payments, per-category margin, a "build kit" action that
  consumes components, and an expected-delivery date on an order.
