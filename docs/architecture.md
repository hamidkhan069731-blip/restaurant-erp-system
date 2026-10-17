# Architecture — RestoERP

Companion to the [README](../README.md). This file covers the decisions that
are expensive to change later: module boundaries, the stock ledger, the offline
sync protocol, and the tenant/isolation model.

---

## 1. Module boundaries

A modular monolith with explicit internal boundaries. Each module owns its
tables and exposes an application service; modules never read each other's
tables directly.

```
modules/
  identity/      users, roles, sessions, audit_log
  catalog/       menu categories, items, modifiers, price versions
  orders/        orders, tickets, payments, shifts
  inventory/     ingredients, recipes, stock_ledger, counts
  procurement/   suppliers, POs, receipts, invoices, matching
  crm/           customers, loyalty, campaigns, reservations
  reporting/     read models, exports, scheduled jobs
  sync/          device registry, command inbox, conflict resolution
```

Rules:

- Cross-module reads go through a query interface, not a join into foreign tables.
- Cross-module writes go through domain events (`order.paid`,
  `goods_received`, `price_changed`) processed in-process.
- If a boundary needs to become a service later, the event stream is already
  the seam.

Extraction order (when it is actually needed): `sync` first (bursty, device
specific), then `reporting` (heavy read load), then `inventory` if multiple
restaurant groups share one deployment.

---

## 2. The stock ledger (the core invariant)

Stock is **never stored as an editable number**. It is the sum of immutable
movements:

```
stock_ledger
  id             uuid pk
  branch_id      uuid
  ingredient_id  uuid
  movement_type  enum   -- sale, purchase, wastage, staff_meal, transfer_in,
                        -- transfer_out, count_adjustment, opening
  quantity       numeric(14,4)   -- signed; sale is negative
  unit_cost      numeric(14,4)   -- weighted average at the time of movement
  reference_type text            -- order_line | goods_receipt | count | ...
  reference_id   uuid
  occurred_at    timestamptz
  created_by     uuid
  idempotency_key text unique
```

Current stock is a materialised view (`stock_balance`) refreshed on movement.
Theoretical usage for a period is derived from sales × recipe. Actual usage is
derived from the ledger. Variance is the difference, and that number — not the
stock count alone — is the food-cost management tool.

**Backdating rule:** movements are never edited. A correction is a new
adjustment movement that references the original. The audit trail therefore
always adds up.

**Cost rule:** moving weighted average. Purchase at a new price updates the
average from that point forward; historical movements keep their recorded cost so
a period's P&L does not change retroactively.

---

## 3. Offline sync protocol

The POS is a first-class offline client, not a browser tab with a cache.

### Device bootstrapping
On login the terminal receives: catalog snapshot (versioned), price list,
tax rules, payment methods, printer config, and a device UUID issued once and
stored locally. It keeps an IndexedDB mirror and a local command queue.

### Command model
Every user action that changes state is a command:

```json
{
  "id": "cmd_01J...",
  "device_id": "dev_...",
  "type": "order.create",
  "idempotency_key": "dev_...:seq:1043",
  "occurred_at": "2026-10-17T12:41:03.221Z",
  "payload": { }
}
```

Commands are appended locally, executed optimistically, and replayed in order
when online. The server rejects duplicate `idempotency_key`s with the previously
stored result — replay is therefore safe.

### Conflict rules

| Data | Rule | Rationale |
|---|---|---|
| Orders / payments | Device wins; server validates totals | The customer has already left with the receipt |
| Stock | Server-side ledger; offline sales enqueue stock movements | Stock is a server projection; multiple terminals must agree |
| Menu availability | Server wins, device re-syncs | One branch's 86 must reach everyone instantly |
| Prices | Server wins on reconnect; device shows an "update required" banner if it fell behind by >1 change | Selling at a stale price is a real loss |
| Shift cash | Device wins for counted cash; server wins for expected cash | Both are needed for reconciliation |

### Reconnection
1. Push queued commands (ordered, idempotent).
2. Pull server changes since last `sync_cursor` (catalog, prices, availability).
3. Show a reconciliation report if anything was rejected; never auto-discard.

### Network detection
`navigator.onLine` is not trusted. The client probes a `/health/sync` endpoint
with a 2 s timeout and backs off to 30 s when offline.

---

## 4. Multi-branch and data isolation

Single database, `organization_id` → `branch_id` on every tenant-scoped row.
Isolation is enforced **in SQL**, not only in application code: each request
sets the session variable `app.branch_ids`, and row-level security policies
filter reads and block writes outside the set. A cashier token for branch A
cannot read branch B even if a query forgets a `where` clause.

Owner and group-accountant roles get a set of branches; branch managers get one.

---

## 5. Money

- Integers in minor units (paisa) for all monetary values.
- Base currency per branch; a reporting currency per organization with an
  exchange-rate table stamped at transaction time.
- Tax rules versioned by date — a tax change never rewrites past receipts.

---

## 6. Reporting model

Reports are not computed on live tables. Nightly (and on demand for the current
shift) a job materialises:

- `fact_sales` — one row per order line, with cost attached from the ledger
- `fact_stock_movement` — daily aggregate by ingredient
- `dim_*` tables for menu item, category, branch, employee, supplier

This keeps month-end and the menu engineering report fast without freezing the
operational database.

---

## 7. Failure modes and their answers

| Failure | Mitigation |
|---|---|
| Internet down | Offline POS + queued commands |
| Server down | POS keeps selling; KDS falls back to printed tickets |
| Printer down | Order can be completed; receipt queued and reprinted later |
| Duplicate order (angry click) | Idempotency key on the command |
| Wrong price pushed | Price versions + audit log + "update required" banner |
| Stock drift | Weekly counts, variance report, adjustment entries with reasons |
| Power cut | POS runs on a UPS; local IndexedDB survives a hard shutdown |

---

## 8. Security

- Server-side RBAC on every endpoint; the UI never decides permissions.
- bcrypt (cost ≥ 12) for passwords; short-lived access tokens with refresh rotation.
- Rate limiting on auth, order creation and payment capture.
- Parameterised queries only; input validation with zod at the boundary.
- Signed, append-only audit log; sensitive fields redacted in logs.
- Backups encrypted at rest; restore rehearsed monthly.
