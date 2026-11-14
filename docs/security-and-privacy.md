# Security and privacy

> **In plain terms.** This document describes what data the system handles, the
> principles that guide how it is protected, and — just as importantly — the limits of
> what the system currently guarantees.
>
> It is deliberately not an implementation review. Details of internal security
> engineering, the findings of internal reviews and the operational credential
> procedures are held separately and are not published here.

**Contents**

1. [What the system handles](#1-what-the-system-handles)
2. [Data residency](#2-data-residency)
3. [Handling principles](#3-handling-principles)
4. [The store identity on every receipt](#4-the-store-identity-on-every-receipt)
5. [Privileged operations](#5-privileged-operations)
6. [The back-office boundary](#6-the-back-office-boundary)
7. [What is not protected](#7-what-is-not-protected)
8. [What this document does not claim](#8-what-this-document-does-not-claim)
9. [Open items](#9-open-items)

---

## 1. What the system handles

The system processes a deliberately narrow set of data. It is worth stating that
narrowness explicitly, because the absence of a data category is the strongest
privacy control available.

| Category | Examples | Personal data? |
| --- | --- | --- |
| **Transaction lines** | Item, quantity, unit price, tax amount, line total | No |
| **Transaction totals** | Taxable value, CGST, SGST, total tax, payable amount | No |
| **Payment mode** | The method recorded against a sale | No |
| **Store identity** | Store name, address, tax registration number, phone number | Business data |
| **Approving account** | The identity of the user who approved an exception | Yes — an employee identifier |
| **Back-office master data** | Item names, item codes, barcodes, selling rates, tax templates | No |

Three categories are notable by their **absence**:

- **No cashier identity is captured.** The till does not record which cashier performed
  a transaction. The one employee identifier the system handles is that of a
  **supervisor who approved an exception** — and only where such an approval occurred.
- **No payment card or bank credentials are captured.** The till records a payment
  mode; it does not take card numbers, authorise transactions or integrate with a
  payment terminal.
- **No demographic or behavioural profile is built.** Nothing in the till joins
  transactions to an identified individual.

The bill model also carries an **optional customer-name field**. The current interface
does not collect a customer name, so that field is not populated by the till; it exists
in the model, is rendered on the receipt if present, and is used as the customer on the
back-office invoice when present. A customer account, loyalty scheme or stored-value
balance does not exist.

---

## 2. Data residency

| Data | Where it resides | Notes |
| --- | --- | --- |
| Bills awaiting reconciliation | The **browser's own database** on the counter machine | Until the bill has been reconciled, the browser is the only copy |
| Reconciled bills | The back office database, **and still retained in the browser's local store** | The application does not delete a bill from the local store once it has been reconciled, so the counter machine keeps a second copy |
| Cached product catalogue | The browser's own database on the counter machine | A copy of master data, not customer data |
| Sample catalogue | The browser's own database on the counter machine | Loaded into an empty item store; see §7 and [`features.md`](features.md) §A.8 |
| Live authorisations | Transient, in the memory of the running till process | Never written to storage; expire automatically; discarded when the bill is closed |
| Authorisation records | The **browser's own database**, stored with the bill | Carries the approving account, the time, the line and the amount — see §5 |
| Store identity | Compiled into the till bundle at build time | Not read from a file at runtime |

The offline-first model has a direct consequence that operators should understand:
**until a bill is reconciled, it exists only on the counter machine.** That is
described in more detail, including its limits, in
[`offline-billing.md`](offline-billing.md).

The distinction between the **live authorisation** and the **authorisation record** is
important and is easy to blur. The live authorisation is transient. The record is not:
it is persisted in the browser's local store, on the same disk as the bills themselves.

---

## 3. Handling principles

These are the principles the implementation is built around. They are stated here
without implementation detail.

| Principle | What it means in practice |
| --- | --- |
| **Fail closed on identity** | A production build that lacks a valid store identity does not start. It shows a configuration-error screen instead of the till. The system cannot silently operate with a placeholder identity. |
| **Identity is configuration, not code** | Store identity is supplied at build time from an untracked configuration source. It is never hard-coded in source, and it cannot be altered on a live workstation by editing a file. |
| **Least privilege at the boundary** | The back office's database and cache are reachable only from inside the container network. Neither is published to the host or the local network. A single published entry point fronts the service. |
| **No credentials in the artefact** | Credentials are read from the environment's configuration source, which is excluded from version control. Credential values are not written into logs, output or image metadata. **This concerns credentials only** — the store's business identity (name, address, tax registration number, phone) *is* compiled into the till bundle by design, because it must appear on the receipt. See §4. |
| **Explicit environment selection** | A deployment must state which environment it is. The tooling does not infer one, and an unset declaration prevents a start rather than defaulting to a development setting. Development-only placeholder values are not accepted outside a local development context. |
| **Available teardown** | The tooling is designed so that a stack can always be stopped and removed, even if its configuration is incomplete or damaged. A misconfiguration cannot leave a deployment running that an operator cannot bring down. |
| **Least data on the counter machine** | The till sends transaction data to the back office. It does not retain a customer account, and it does not write credentials to storage. |
| **Test isolation** | The automated test suites run entirely in throw-away temporary directories, use synthetic values only, never read a real configuration file, never require a network, and clean up after themselves even when a test fails. |

---

## 4. The store identity on every receipt

The store's name, address and tax registration number appear on every printed
receipt. Three properties of this are worth recording:

1. **The tax registration number is validated before the till will run.** The
   value is checked against the expected format, and generic development
   placeholders are treated as "not configured" for production purposes. A
   production build with a missing or placeholder identity refuses to start.
2. **The checksum digit of the registration number is not independently
   verified.** The format check establishes shape, not validity with the tax
   authority.
3. **The identity is business information, not a credential.** It is embedded in
   the build deliberately, so that the receipt carries the correct store. It is
   visible to anyone who handles a receipt, and it is not protected by any
   secrecy control.

This is a configuration-integrity control, not a tax-authority integration. The
compliance review of the receipt format, and of the tax treatment of deposit lines, are
matters for the deploying organisation and their tax adviser and are not settled here.

---

## 5. Privileged operations

A small number of till operations cannot be completed by a cashier alone. For those
operations the till requires a supervisor's credentials, verified against the back
office at the moment the operation is attempted.

Three properties are deliberately built in:

| Property | Reason |
| --- | --- |
| The authorisation is verified at the point of use | A decision accepted from an earlier cached state cannot reflect a revoked account or a changed role assignment |
| The **live authorisation** is transient | It is held in memory only, so it cannot be read back from disk or survive a restart; it expires on its own and is discarded when the bill is closed |
| The **authorisation record** is stored with the bill | The back office holds no record of who approved an operation, so the evidence must travel with the bill |

The second and third rows point in opposite directions, and both are accurate: the
**live authorisation** is transient, while the **record** of what was approved is
persisted in the browser's local store. That record contains the approving account's
identity, so it is personal data at rest on the counter machine, and it is subject to
the same protections — or the same absence of protections — as the bills themselves.

Which specific operations require authorisation, how long an authorisation remains
valid, and the approval policy itself are governance matters for the deploying
organisation. The accepted risk position and the compensating controls for a
supervisor's credentials being entered at a shared counter terminal, and for the
authorisation record being held on the machine, are matters for the deploying
organisation too. None are stated here.

---

## 6. The back-office boundary

All communication between the till and the back office travels over the back
office's documented REST interface.

The integration surface is deliberately narrow. The till can:

- sign in to, and sign out from, the back office;
- read the product catalogue, the prices for a price list, the barcodes and the tax
  templates;
- read the **sales invoices** it has created, in order to check whether one already
  carries a given bill's key;
- read the account and role information of the signed-in user;
- create a sales invoice.

It has no other back-office capability. A bill pushed to the back office is
de-duplicated by matching a unique per-bill identifier, which is intended to prevent
a retried bill from being invoiced twice — see
[`offline-billing.md`](offline-billing.md) §3 for the limits of that mechanism.

**On sessions.** The sign-in operation is exercised as part of a supervisor
authorisation. The till does not present a cashier sign-in screen, and it holds no
embedded or bypassed administrative credential of its own. The password entered at the
authorisation prompt is not written to storage by the application; the **session** that
results from the sign-in is held by the browser for the life of that session, in the
ordinary way for a browser talking to a web application. Because that session persists
for its lifetime, it is worth noting that the session established by an authorisation is
the session subsequent back-office calls are made under. **Not yet established** — how a
till session is established for ordinary billing, and which account bills are posted
under, has not been documented for this project.

**On what reaches the back office.** The invoice created for a bill carries the bill
identifier, the idempotency key and the offline-origin marker. The **authorisation record
is not currently transmitted**, so the back office holds no record of who approved an
operation — see §5 and §7.

The production network arrangement, the point at which encryption is terminated, the
certificate arrangement and the access-control policy for the back office have not been
finalised for this project and are not documented here.

---

## 7. What is not protected

A security document that lists only protections is not useful. The following are
**not** provided by the system as it stands.

| Area | Current position |
| --- | --- |
| **Local bill data at rest** | The browser's local database is not encrypted by the application. It inherits whatever the operating system and the browser profile provide. The same applies to the authorisation records stored with the bills. |
| **Local bill backup** | The application does not back up the local bill queue. A cleared browser profile or a rebuilt workstation loses any bill that has not yet been reconciled. |
| **Local bill retention** | Reconciled bills are not removed from the browser's local store by the application, and there is no retention or compaction policy. The workstation therefore accumulates a copy of every bill it has taken. |
| **Authorisation records at rest** | The authorisation record — including the approving account's identity — is held in the same unencrypted local store as the bills, with the same absence of backup and retention control. |
| **Cash accountability** | The till performs **no** cash counting, drawer reconciliation, shift management or day-end close. Cash in the drawer is not reconciled by this system. |
| **The sample catalogue in production** | A sample catalogue of twelve sporting-goods lines is loaded into an empty local item store in **any** build, and is not covered by the production identity guard. Those lines are sellable until an import replaces them. See [`features.md`](features.md) §A.8. |
| **Print output** | A printed receipt is a physical artefact carrying the store's tax registration number. Its custody, disposal and the treatment of such receipts are outside the system. |
| **Session handling at the workstation** | The system does not lock the workstation, does not enforce a per-user browser profile and does not control physical access to the counter. |
| **Audit log** | There is no separate, tamper-evident audit trail of till activity. The back office's standard document history applies to the invoices it holds — and it currently holds **no authorisation event**, because the authorisation record is not transmitted. |
| **Transport security** | Encryption at the web endpoint is not configured in the current implementation, and the production transport arrangement has not been decided. See [`deployment-overview.md`](deployment-overview.md). |
| **Workstation hardening** | Operating-system configuration, disk encryption, screen lock, endpoint protection and physical security of the counter are outside the application's scope. |

The accepted risk position for local bill data and for the authorisation records held
with it, and the compensating controls (disk encryption, browser-profile protection,
workstation backup, retention, cash-handling procedure, and physical security of the
counter) are matters for the deploying organisation and are not stated here.

---

## 8. What this document does not claim

- It does **not** claim a penetration test, security audit or certification. No
  independent assessment of this system has been carried out.
- It does **not** claim a formal privacy impact assessment or a data-protection
  compliance review. Whether the handling described here satisfies a particular
  statutory regime is a question for the deploying organisation and their advisers.
- It does **not** claim that the system is free of defects or weaknesses. The
  limitations in §7 are stated so that no such impression is created.
- It does **not** describe the internal security review, its findings or the
  remediation work. Those are held separately.
- It does **not** claim a deployment in any environment. See
  [`deployment-overview.md`](deployment-overview.md).
- It does **not** claim that a bill is invoiced exactly once. See
  [`offline-billing.md`](offline-billing.md) §3.

---

## 9. Open items

**Not yet established** — the following are not held by the author of this document and
are needed to complete it:

1. the deploying organisation's data-protection and retention requirements;
2. the agreed retention periods for bills, receipts and authorisation records;
3. the agreed position on local bill data, and on the authorisation records held with
   it, at the counter machine;
4. the cash-handling and drawer-reconciliation procedure;
5. the security-disclosure contact and the responsible-disclosure process;
6. the agreed policy for physical custody of printed receipts;
7. the decision on whether the sample catalogue should be excluded from production
   builds.

A contact address for security disclosures, correctness reports and licensing enquiries
has not been designated.

---

## Related documents

- [`overview.md`](overview.md) — the problem, the scope and the people who use the system.
- [`architecture.md`](architecture.md) — where each kind of data lives.
- [`offline-billing.md`](offline-billing.md) — the local bill store and its limits.
- [`deployment-overview.md`](deployment-overview.md) — how the back office is packaged.
- [`roadmap.md`](roadmap.md) — the operational work that has not been started.

---

*Documentation for the Sports Store Management System.*
