# Roadmap — RestoERP

Each phase is shippable on its own and has an exit test that a restaurant
manager — not a developer — can judge. Phases are sequential; everything in a
"Not yet" row stays out until the phase above it is genuinely done.

---

## Phase 0 — Foundation

- Repository, TypeScript strict mode, lint/format, CI running typecheck + tests
- Schema design and migrations for identity, catalog, branches, terminals
- Auth: login, refresh rotation, bcrypt, roles, audit log scaffold
- Branch / terminal model with SQL row-level isolation
- Seed script: one branch, 20 menu items, 5 roles

**Exit test:** a cashier logs into branch A; a manager cannot use branch B's data.

---

## Phase 1 — POS and KDS (online)

- Table map, order create/edit, modifiers, split and merge
- Payment capture: cash, card, wallet, credit
- Receipt generation and ESC/POS printing
- KDS tickets with aging, station routing, 86 (sold-out) toggle
- Shift open/close with cash drawer

**Exit test:** run one full service — every order reaches the kitchen, every
bill prints, the shift closes with a counted drawer.

---

## Phase 2 — Offline POS and sync

- Catalog + price snapshot to IndexedDB; device registration
- Command queue with idempotency keys and ordered replay
- Conflict rules from [architecture.md](architecture.md) §3
- Reconciliation screen on reconnect
- "Stale price" banner

**Exit test:** unplug the network mid-service, take 10 orders and 3 payments,
restore the connection — all 10 orders and 3 payments appear exactly once, and
the stock movements match.

---

## Phase 3 — Inventory and recipes

- Ingredient master, units and conversions, sub-recipes, yield %
- Recipe editor on the menu item with live cost calculation
- Stock ledger and `stock_balance` view
- Sales → recipe → stock movement pipeline
- Wastage, staff meals, transfers
- Stock count with variance report (theoretical vs. actual, priced)

**Exit test:** sell 1,000 of one item; theoretical usage matches the recipe
exactly, and a deliberate 200 g wastage entry shows up in the variance report.

---

## Phase 4 — Procurement

- Supplier records and scorecards
- Requisition → PO → goods receipt → supplier invoice
- Three-way match with tolerances; discrepancy queue
- Landed cost (freight, duty) into the moving average
- Expiry alerts for batch-tracked items

**Exit test:** raise a PO, receive 10% short at a price 5% higher, and the system
flags it, records the actual received quantity, and updates the moving average
correctly.

---

## Phase 5 — CRM, loyalty, WhatsApp

- Customer records linked to orders
- Points and tiers with an expiry policy
- Opt-in management; WhatsApp Business API template campaigns
- Reply handling with a simple inbox
- Reservations and waitlist

**Exit test:** an opted-in customer receives a campaign, replies, and the reply
is visible in the inbox; an opted-out customer is never messaged.

---

## Phase 6 — Multi-branch analytics

- Nightly fact tables; menu engineering matrix
- Branch and group P&L by shift, day, week
- Prime cost, food cost %, labour % trends
- Discount/void leakage report

**Exit test:** month-end P&L for two branches reconciles to the POS totals with
zero spreadsheet work.

---

## Not yet (deliberate)

- Native mobile apps — the responsive manager view is enough until proven otherwise
- AI forecasting of demand — valuable, but only after 6 months of clean sales
  history exists; forecasting on bad data is worse than a par-level rule
- Microservices — extract only when a module's load profile actually demands it
- Franchise royalty accounting — different business, different product
- Delivery aggregator integrations (Foodpanda etc.) — requires commercial
  agreement; the order model is designed to accept them later

---

## Definition of done for every phase

1. Migration reversible and tested on a copy of the seed database.
2. Integration tests for the money paths (order → payment → receipt → ledger).
3. Audit log entries for every mutation that touches money, stock or permissions.
4. README updates in the same commit, not later.
5. A demo path that works from a clean clone in under five minutes.
