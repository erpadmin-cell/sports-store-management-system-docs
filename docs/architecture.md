# Architecture

> **In plain terms.** The system is two applications with a narrow seam between them.
> The till application handles the counter; the back office holds the records. This
> document names the parts, shows how they fit together, and walks through the three
> workflows that matter: an ordinary sale, a sale taken without a connection, and a
> sale that needs a supervisor's authorisation.

---

## 1. Shape of the system

```
┌──────────────────────────────────────────────────────────────────────┐
│  COUNTER MACHINE                                                    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Till application (runs in a web browser)                      │  │
│  │                                                                │  │
│  │   Scan item ──► Build basket ──► Show totals ──► Checkout     │  │
│  │                                  │                             │  │
│  │                  ┌───────────────┴──────────────┐              │  │
│  │                  ▼                              ▼              │  │
│  │             Print receipt              Ask a supervisor         │  │
│  │           (80 mm paper)                  to approve             │  │
│  │                  │                              │              │  │
│  │                  ▼                              ▼              │  │
│  │           Local bill store ◄──── approval record kept with    │  │
│  │        (the browser's own database)        the bill            │  │
│  │                                                                │  │
│  │   Reconciliation process ───────► REST interface ──────────┐   │  │
│  └───────────────────────────────────────────────────────────┼───┘  │
└───────────────────────────────────────────────────────────────┼──────┘
                                                                │
                              ┌─────────────────────────────────▼───────┐
                              │  BACK OFFICE  (a set of containers)      │
                              │                                          │
                              │   web endpoint ──►  ERPNext v15 API      │
                              │                   │                      │
                              │                   ├──►  MariaDB            │
                              │                   ├──►  Redis (queue)      │
                              │                   │                      │
                              │                   └──►  workers,           │
                              │                        scheduler          │
                              └──────────────────────────────────────────┘
```

*Diagram authored from the implementation for documentation purposes. It should be
reviewed, and converted into a branded asset, before publication.*

---

## 2. The components

### 2.1 Till side

| Component | Responsibility |
| --- | --- |
| **Barcode entry** | Captures scanner input, resolves it to an item, falls back to a name search |
| **Basket** | Renders the live basket; owns line editing (quantity, rate, removal) |
| **Totals** | Taxable value, CGST, SGST, total tax and grand total for the basket as a whole |
| **Checkout** | The checkout actions: print and reconcile now, or save locally for later reconciliation — plus the payment-mode and deposit-line controls |
| **Receipt** | Renders the 80 mm thermal receipt and drives the print stylesheet |
| **Authorisation request** | Collects a supervisor's credentials for an operation that requires authorisation |
| **Sync indicator** | Shows connectivity and the number of bills awaiting reconciliation |
| **Header** | Store identity, sync state and the master-data import action |
| **Configuration-error screen** | Blocking screen shown when the build lacks a valid store identity |

### 2.2 Shared till logic

| Concern | Responsibility |
| --- | --- |
| **Basket arithmetic** | Line totals, basket totals, tax accumulation — no I/O |
| **Tax arithmetic** | Intra-state tax split, rounding-safe — no I/O |
| **Configuration validation** | Validates the build-time store identity, including the registration-number format — no I/O |
| **Back-office access** | The entire HTTP surface to the back office, in one place |
| **Local store** | The browser database schema and its queries |
| **Sample data** | A small local catalogue loaded when the local item store is empty |
| **Reconciliation process** | The loop that drains pending bills |
| **State hooks** | Bind the above to the interface |

The separation is deliberate: the basket arithmetic, the tax arithmetic and the
configuration validation are **pure functions with no I/O**, which is what makes them
straightforward to reason about and safe to change.

### 2.3 Back office

Standard ERPNext v15, extended by this project with three additional sales-invoice fields
(offline bill identifier, idempotency key, offline-origin marker) so that invoices
produced by the till can be reconciled.

---

## 3. Workflow 1 — an ordinary sale

1. The cashier scans an item; it is priced from the locally cached catalogue and added
   to the basket.
2. Tax is computed per line as an intra-state split, rounded so the halves re-add exactly.
3. On checkout, the payment mode is selected and the bill is written to the **local store**
   first, marked *pending*.
4. The receipt is rendered and printed.
5. The bill is then posted to the back office. The client **checks for an existing
   invoice with the same idempotency key first**, and only then creates one.
6. The bill is marked *synced* and the pending count decreases.

Step 3 before step 5 is what makes the workflow safe: the local write is the commit point
for the cashier, and the network is an optimisation, not a dependency.

## 4. Workflow 2 — a sale taken without a connection

1. Connectivity drops; the sync indicator shows the offline state.
2. Steps 1–4 of the ordinary workflow proceed unchanged — the local store is
   authoritative.
3. The bill stays *pending*.
4. When connectivity returns, the reconciliation process drains the pending queue in
   order.

The queue is drained with the same de-duplication contract as an ordinary sale, so a
bill that was partly transmitted when the link failed is reconciled against the same
invoice on retry rather than a second one. The limits of that contract are set out in
[`offline-billing.md`](offline-billing.md) §3.

## 5. Workflow 3 — a supervised exception

1. The cashier attempts an operation that requires authorisation.
2. The request opens and the cashier asks a supervisor to approve it.
3. The supervisor's credentials are verified against the back office **at that moment**.
   Nothing is accepted on the strength of an earlier cached decision.
4. On success, an authorisation is created **in memory**, scoped to that operation and
   that line, with an expiry.
5. The cashier completes the operation. The **authorisation record** — who approved, when,
   which line and amount — is captured on the bill and stored with it in the local store.
6. The live authorisations are discarded when the bill is closed, and they expire on
   their own regardless.

**Not yet established** — the authorisation record is captured on the bill and stored
locally, but it is **not currently transmitted to the back office**. The invoice created
there carries the bill identifier, the idempotency key and the offline-origin marker; it
does not carry the approver. Whether it should, and by what mechanism, has not been
decided.

The specific operations that require authorisation, the authorisation duration, and the
authorisation policy are governance matters for the deploying organisation and are not
stated here.

---

## 6. Design decisions worth knowing

| Decision | Reason |
| --- | --- |
| The local store is written **before** the network call | The cashier's sale must never depend on connectivity |
| De-duplication is enforced by **checking before creating**, rather than by a database constraint | The bill must be reconciled by identity, so that a retried bill maps to one invoice |
| The tax split is computed on the client | The receipt must be correct even when there is no back office to ask |
| Rounding assigns the remainder to one side | A receipt whose two tax halves do not re-add to the tax total is a defect |
| Store identity is build-time configuration | A tax registration number must never be hard-coded in a source file, and a production build with an invalid identity must refuse to start |
| The live authorisation is held in memory only | An authorisation that can outlive the process, or be read back from disk, is not a live authorisation |
| The authorisation **record** is stored with the bill | The back office has no record of who approved an operation, so the evidence must travel with the bill |
| The HTTP surface to the back office lives in one place | One place to review for what the till can and cannot do |

---

## 7. Data boundaries

| Data | Where it lives | Notes |
| --- | --- | --- |
| Product catalogue, prices, tax templates | Back office, cached locally in the browser | Refreshed on demand by the cashier; the import reads a single page of results — see [`features.md`](features.md) §A.5 |
| Sample catalogue | The browser's local store | Loaded into an empty item store, in any build — see [`features.md`](features.md) §A.8 |
| Bills | Browser store, then back office | The browser copy is the offline queue, and is retained after reconciliation |
| Live authorisations | Application memory only | Not persisted; expire automatically |
| Authorisation records | Stored with the bill in the browser store | Carries the approving account, the time, the line and the amount |
| Credentials entered at the till | Held in the form only, and sent to the back office for verification | The password is not written to storage by the application. The back-office **session** that results from the sign-in is held by the browser for the life of that session. |
| Tax registration number, address, phone | Build configuration | Not stored in the source tree |

---

## 8. Scale characteristics

**Not independently verified** — no load, concurrency or transaction-volume
measurements have been taken for this system, and none are claimed. The following design
facts bound the architecture but do not substitute for measurement:

- the till is a single-screen, single-cashier interaction;
- the offline queue is bounded by the browser's storage quota, and no compaction or
  queue-limit policy has been implemented;
- the reconciliation loop polls on a timer and drains the queue in order, so a large
  backlog is drained serially;
- the de-duplication contract assumes a single writer, which bounds the design to one
  counter per back office;
- the item import reads a single page of results, which bounds the catalogue size the
  till can hold.

Any statement about throughput, latency or scale would need benchmarking that has not
been carried out.

---

## 9. Extending the system

The following are integration points the architecture supports. Each would need a
decision before it is built:

- **Additional payment modes** — the payment mode is already a first-class field on a
  bill; integrating a terminal means adding a settlement step, not changing the data model.
- **A second till or a second store** — the design assumes one store in one tax region;
  a second store would require per-store configuration and a per-store tax jurisdiction.
  Note that the de-duplication contract in §3 of `offline-billing.md` also assumes a
  single writer, and would need revisiting before a second till is introduced.
- **A customer-facing channel** — the back office already exposes the product catalogue
  over its REST interface, but a public catalogue needs its own read model and its own
  pricing rules.
- **Paginated import and till-side stock** — see [`features.md`](features.md) §A.5 and
  §C for the present limits and the candidate directions.
