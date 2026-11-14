# Sports Store Management System

**A point-of-sale system for a sporting-goods retailer, designed to keep selling when the internet connection drops.**

This repository is the **public documentation** for the project. It contains
documentation only — no application source code, no credentials, and no business data.

> The application itself is **closed-source** and is developed and operated separately.
> This repository deliberately contains **no source code**, no configuration templates
> and no operational credentials.
> See [`NOTICE.md`](NOTICE.md) for the full statement.

---

## 1. The problem this solves

Retail counters rarely have a perfect internet connection. A connection can drop in the
middle of a busy afternoon, or a store can be located where the signal is poor. Most
point-of-sale systems respond in one of two ways when that happens:

- they **refuse the transaction**, so the till stops selling; or
- they **lose the transaction**, so the sale happened at the counter but never reached
  the records.

Both are expensive. The second is worse, because the shop does not know it has sold
something.

This system takes a third approach: the sale is **committed on the till first** and
sent to the central records afterwards, when the connection returns.

---

## 2. Who benefits

| Role | What the system gives them |
| --- | --- |
| **Cashier** | A fast, single-screen counter screen: scan an item, see the price and tax immediately, take payment, print the receipt. No waiting on the network to complete a sale. |
| **Supervisor** | Control over the few operations a cashier is not allowed to complete alone, with the authorisation checked against the central records at the moment it is requested. |
| **Back office / finance** | One authoritative record of every sale, whether it was taken with a working connection or without one, with the tax split on the receipt matching the record. |

---

## 3. How it works, in simple steps

The system has two parts that meet at the counter.

**The till application** is the screen the cashier uses. It runs in a web browser on the
counter machine and stores its own working copy of the catalogue and its bills in the
browser's local storage — a small database built into the browser itself.

**The back office** is a shared business-management system that holds the product
catalogue, the prices, the tax rules and the authoritative record of every sale. It is
the same system that would run the shop's accounts, stock and reporting.

The flow a cashier actually experiences is short:

1. **Scan.** An item's barcode is scanned into the counter screen. The price and tax
   come from the catalogue already stored on the till, so this works with no connection.
2. **Build the basket.** Items are added to a running bill. Tax is split into its two
   government halves and shown on screen as it happens.
3. **Pay and print.** The cashier takes payment and chooses how to complete the sale.
   An 80 mm receipt prints immediately from the till's own copy.
4. **Reconcile.** If the connection is down, the bill waits in the till's queue. When
   the connection returns, a background process sends it to the back office and marks it
   as done.

The interesting part is step 4, and it is explained properly in
[`docs/offline-billing.md`](docs/offline-billing.md).

---

## 4. What the system can do today

The following are **implemented**. Each one is described in more detail in
[`docs/features.md`](docs/features.md).

- **Fast counter operation** — barcode and item-code lookup, a live basket, and a
  checkout that prints an 80 mm thermal receipt.
- **Correct Indian GST on the receipt** — every taxable line is split into its central
  and state government halves, rounded so the two halves always add back exactly to the
  tax shown.
- **Selling with no connection** — a bill is written to the till's own storage before
  any attempt to reach the back office, so a dropped connection never loses a sale.
- **Automatic reconciliation** — queued bills are sent to the back office when the
  connection returns, and each one carries a unique reference so the same bill is not
  recorded twice.
- **Mixed baskets** — regular sales, hired goods billed by the day, and refundable
  deposits held as a liability can all appear on one bill.
- **Supervised exceptions** — a small number of operations require a supervisor's
  authorisation, verified live and expiring automatically.
- **Correct store identity on every receipt** — the store's details are supplied when
  the software is built, and a production build that is missing them refuses to start
  rather than printing a blank tax registration number.

---

## 5. What is built but not yet verified in a live environment

This is an important distinction, and the documentation keeps it visible throughout.

**No deployment of this system exists in any environment.** It has not been installed
at a store, no sale has been completed against a live back office, and no receipt has
been printed on physical paper.

| Capability | What is outstanding |
| --- | --- |
| Bringing up the back-office stack and creating the site | The configuration and scripts exist; the stack has not been started end-to-end |
| Installing the GST application | The installer exists; it has not been run |
| Creating the supervisory account | The setup script exists; it has not been run |
| Creating the three extra invoice fields used for reconciliation | The script exists; it has not been run |
| Printing a receipt on real paper | The layout and print stylesheet exist; no hardware print test has been done |
| A bill travelling to the back office and appearing as an invoice | The code and the de-duplication logic exist; the round trip has not been performed |
| Importing a product catalogue of realistic size | The import exists; it has not been run against a live system |

**Not yet established** — an independent verification report, a requirements baseline
and formal acceptance criteria are not part of this documentation set. The inventory in
[`docs/features.md`](docs/features.md) is therefore derived from the implementation
itself and has not been reconciled against a signed specification.

---

## 6. Known limitations today

These are real and are stated plainly rather than buried.

| Limitation | What it means in practice |
| --- | --- |
| **The queue of unsent bills lives only on the counter machine** | Until a bill is reconciled, the browser's own storage is the only copy. It is not encrypted beyond what the operating system provides, is not backed up by the application, and is lost if the browser profile is cleared |
| **Reconciled bills are not deleted locally** | The counter machine keeps a second copy of every bill it has taken, indefinitely |
| **No cash counting or shift close** | The till records transactions and a payment mode. It does not count a drawer, manage a shift, close a session, or reconcile cash — that remains a manual activity |
| **De-duplication assumes a single writer** | The protection against recording the same bill twice relies on one till writing to the back office at a time. A second till against one back office would break that assumption |
| **The product import reads a single page of results** | A catalogue larger than the import limits is silently truncated, and the till simply holds fewer items |
| **A small sample catalogue is loaded when the local catalogue is empty** | This happens in every build, including production. Those sample lines are sellable until a real import replaces them |
| **No till-side stock tracking** | Stock quantities are not brought in from the back office, and the till keeps no count of its own |
| **The authorisation record is not sent to the back office** | The back office holds no record of who authorised an operation, because that information currently stays with the bill on the counter machine |
| **No discounts** | There is no markdown, coupon or percentage-reduction capability in this release |
| **Transport encryption is not configured** | The back office is currently reached over an unencrypted connection; the production arrangement has not been decided |

The full list, with the reasoning behind each, is in
[`docs/security-and-privacy.md`](docs/security-and-privacy.md) §7 and
[`docs/offline-billing.md`](docs/offline-billing.md) §7.

---

## 7. What could be considered in future

These are **ideas for future development**. None of them is built, none is scheduled, and
none is a commitment. The current system remains a single-store installation on the
retailer's own hardware.

| Direction | What it would involve |
| --- | --- |
| **Cloud hosting** | Moving the back office from the shop's own hardware to a managed environment, which would also require the network, encryption, backup, monitoring and access-control arrangements that are currently undecided |
| **Multiple stores on one installation** | Several shops sharing one back office, each with its own catalogue, prices and tax jurisdiction. The current configuration assumes one store in one tax region, and the de-duplication logic assumes one writer |
| **Centralised administration** | One place to update the counter software, with a defined delivery, versioning and rollback process — none of which exists today |
| **Improved inventory management** | Bringing real stock levels in from the back office and letting the till track what it has sold |
| **A hosted subscription service** | Turning the product into a service the retailer pays for, which would raise its own commercial, hosting and data-handling questions |
| **Additional integrations** | Card and bank payment settlement, customer accounts and stored value, returns handled at the counter, and inter-state tax handling — each currently excluded by design |
| **Better testing** | Automated tests for the counter application itself, which today has no test framework and relies on static type checking |

The reasoning behind each, and what would have to change first, is in
[`docs/roadmap.md`](docs/roadmap.md).

---

## 8. The documentation

| Document | What it covers |
| --- | --- |
| [`docs/overview.md`](docs/overview.md) | The problem, the scope, the people who use it, and what is deliberately out of scope |
| [`docs/technology-stack.md`](docs/technology-stack.md) | Every technology used, what each one is for, and how it is versioned |
| [`docs/features.md`](docs/features.md) | What is implemented, what is implemented but unverified, and what is out of scope |
| [`docs/architecture.md`](docs/architecture.md) | The components, how they talk to each other, and the three main workflows |
| [`docs/offline-billing.md`](docs/offline-billing.md) | How selling without a connection works, and the limits of the protection against duplicates |
| [`docs/deployment-overview.md`](docs/deployment-overview.md) | How the system is packaged and operated |
| [`docs/security-and-privacy.md`](docs/security-and-privacy.md) | The data the system handles, how it is protected, and what it does *not* claim |
| [`docs/roadmap.md`](docs/roadmap.md) | The current boundaries and the possible directions ahead |

### Technology at a glance

Each entry states what the component is *for*, not just what it is.

| Layer | Technology | What it does here |
| --- | --- | --- |
| Counter application | React and TypeScript, built with Vite, styled with Tailwind CSS | Builds the single screen the cashier uses; the type system keeps the different states of a bill from drifting apart |
| Offline storage | Dexie, over IndexedDB | The browser's own small database, which holds the catalogue and the queue of bills awaiting reconciliation |
| Back office | ERPNext v15 with its India Compliance app | Holds the catalogue, prices, tax rules and the authoritative record of every sale |
| Database | MariaDB | Stores the back-office records |
| Cache and background queue | Redis | Keeps repeated lookups fast, and carries the back office's background work |
| Packaging | Docker Compose | Describes the back office as a set of separate programs that can be started and stopped together |

Full detail, including exact versions and how they are pinned, is in
[`docs/technology-stack.md`](docs/technology-stack.md).

---

## 9. How to read this documentation

These documents are written to be **verifiable**. Where a capability is implemented but
has not yet been exercised end-to-end, it is labelled as such rather than presented as
working software. Where the author does not have the information, it is stated as
**not yet established** rather than estimated.

| Label | Meaning |
| --- | --- |
| *(no label)* | Implemented, with the supporting evidence cited in the document |
| **Not yet verified** | Implemented in the code, but never exercised end-to-end in a live environment |
| **Not yet established** | The information is not available to the author; the section states what is needed to complete it |
| **Future** | A candidate for future development. Not built, not scheduled, not a commitment |

Two conventions are used throughout. Terms such as *the client*, where they appear in
technical passages, mean **the software making the request** — not a business or
organisation. And where a document mentions what has **not** been decided, it says so
rather than filling the gap with an assumption.

---

## 10. Contributing, security and contact

- A contribution guide, a code of conduct, a security-disclosure policy and a contact
  address are **not yet in place**. They are an internal publication decision and are
  not covered by this documentation set.
- **No licence file is published** with these documents. The terms for reuse have not
  been decided; see [`NOTICE.md`](NOTICE.md). The application remains closed-source
  regardless.

---

*Documentation for the Sports Store Management System.*
