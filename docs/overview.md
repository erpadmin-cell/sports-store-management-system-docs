# Overview

> **In plain terms.** This system is a point-of-sale application for a sporting-goods
> retailer. It lets a cashier scan items, build a bill, take payment and print a
> receipt — and it keeps working when the internet connection does not. Bills taken
> without a connection are sent to the central back office automatically once the
> connection returns.
>
> This document introduces the problem, the shape of the solution, who uses it, and
> what is deliberately not included.

---

## 1. The problem

A sporting-goods retailer sells a high mix of items — equipment, apparel, footwear,
accessories — across a busy counter. Three things make that counter hard to serve with
off-the-shelf software:

1. **The connection is not dependable.** Retail locations routinely have intermittent or
   absent connectivity. A till that refuses to sell when the network is down is a till
   that loses sales.
2. **The tax treatment is not generic.** Indian retail invoicing demands an
   intra-state CGST + SGST split on the receipt itself, and a deposit on hired or
   returned goods has its own treatment that is easily applied incorrectly.
3. **The back office is powerful but not shaped for a counter.** A full business
   management system is excellent as a back office and awkward as a rapid counter
   checkout.

**What the tax terms mean.** *GST* is India's consumption tax. For a sale within a
single state, the tax is split into two equal halves — a central government share
(*CGST*) and a state government share (*SGST*) — and both appear separately on the
receipt. A *security deposit* is a refundable amount held against hired or returned
goods; it is a liability rather than a sale, which is why it needs its own treatment.

## 2. The approach

The system is split into two parts that meet at the till:

- a **till application** — a focused, single-screen web application built for the
  cashier, which keeps its own working copy of the catalogue and its bills in the
  browser's local storage;
- a **back office** — ERPNext v15 with the India Compliance app, holding the product
  catalogue, pricing, tax templates and the authoritative record of each sale.

The till is deliberately **not** a general business-system client. It does a small
number of things extremely well — capture a sale, price it correctly, print it, and
reconcile it — and delegates everything else.

## 3. Who uses it

| Role | What they do |
| --- | --- |
| **Cashier** | Scans items, builds a bill, takes payment, prints the receipt |
| **Supervisor** | Authorises exceptions that a cashier cannot perform alone. The authorisation is verified against the back office at the moment it is requested, and the live authorisation expires automatically. The specific operations that require authorisation are a matter for the deploying organisation and are not enumerated here. |
| **Back office / finance** | Maintains the product catalogue, prices and tax templates; reviews the invoices the till produced, whether online or deferred |

## 4. Scope of the first phase

| Dimension | Scope |
| --- | --- |
| Stores | One |
| State / tax jurisdiction | One — intra-state supply only (CGST + SGST) |
| Channels | Single counter till; no customer-facing web or mobile channel |
| Currency | One (Indian Rupee) |
| Payment capture | Recorded against a payment mode; **card and bank settlement are not integrated** — reconciliation happens in the back office |

Two consequences follow, and they are worth stating plainly because they shape the
design:

- **There is no inter-state (IGST) path.** Selling to a customer in another state is out
  of scope for this phase and would need tax handling that the current implementation
  deliberately does not perform.
- **The treatment of a security deposit is a configuration decision.** Whether the
  deposit line is taxed at all, and at what rate, is set per deployment. The specific
  rates are a commercial and tax matter for the deploying organisation and their tax
  adviser, and are **not published here**.

## 5. Offline-first, in plain terms

**"Offline-first" means the till can complete and print a sale with no network
connection at all**, subject to the limitations set out in
[`offline-billing.md`](offline-billing.md).

The till does not treat the network as a precondition:

1. The cashier completes the sale. The bill is written to the till's own storage and
   marked as pending.
2. The receipt is printed immediately, from that local copy.
3. A background process watches for connectivity. When the link returns, pending bills
   are sent to the back office.
4. Each bill carries a **unique idempotency key** — a one-off reference number. Before
   creating an invoice the till checks whether one already exists for that key, so a
   bill retried after a dropped connection is normally reconciled against the same
   invoice rather than a second one.

**What step 4 does and does not guarantee.** The check is a query followed by a create,
and it is **not** a single atomic operation. The guarantee therefore rests on an
assumption that there is only one writer at a time — which holds for a single counter
till, and is the design this project assumes. It does not hold for a second till
writing to the same back office, and it is not backed by a uniqueness constraint in the
database. **Not independently verified** — the circumstances under which a duplicate could
still occur have not been formally analysed or tested. The mechanism, its limits and
the assumptions behind it are described in
[`offline-billing.md`](offline-billing.md).

## 6. Deliberately out of scope

The following are **not** part of this system, and the documentation does not imply
otherwise:

- customer accounts, loyalty programmes and stored-value balances;
- purchase, stock-receipt and stock-transfer workflows (these live in the back office);
- returns and refunds as a counter-side workflow;
- **shift management, cash counting and day-end close** — the till records transactions
  and payment modes; it does not count a drawer, manage a shift or close a session, and
  it performs **no cash reconciliation of its own**;
- electronic invoicing / IRN generation;
- printing of A4 or statutory invoice formats (the till prints an 80 mm thermal
  receipt; the back office produces the formal document);
- multi-store consolidation and multi-warehouse stock;
- any customer-facing web or mobile storefront;
- card, UPI or bank payment **settlement** integration — the till records the payment
  mode, the back office reconciles;
- **discounts, markdowns and coupons** — the bill model has no field for a reduction.

## 7. Requirements and business rules

**Not yet established**

The governing requirements specification, the acceptance criteria and the signed
business-rule set are not part of this repository. The feature inventory in
[`features.md`](features.md) is therefore derived from the implementation and has not
been reconciled against an approved requirements baseline.

To complete this section, the following would be needed:

1. the current requirements specification and its version history;
2. the accepted statement of scope for the first phase;
3. the agreed treatment of deposits, write-offs and held stock;
4. the acceptance criteria the system is expected to meet.

## 8. Where to go next

- [`technology-stack.md`](technology-stack.md) — what it is built on, and why.
- [`features.md`](features.md) — implemented, implemented-but-unverified, and out of scope.
- [`architecture.md`](architecture.md) — components and workflows.
- [`offline-billing.md`](offline-billing.md) — how selling without a connection works.
- [`deployment-overview.md`](deployment-overview.md) — how it is packaged and operated.
- [`security-and-privacy.md`](security-and-privacy.md) — data handling and protections.
- [`roadmap.md`](roadmap.md) — the current boundaries and the possible directions ahead.
