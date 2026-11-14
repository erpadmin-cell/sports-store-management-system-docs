# Features

> **In plain terms.** This document lists what the system actually does. It separates
> three things that are routinely conflated:
>
> - **A. Currently implemented** — built and working in the code.
> - **B. Implemented but not independently verified** — built, but never exercised
>   end-to-end in a live environment.
> - **C. Potential future improvements** — ideas only. Nothing in this section is
>   built, scheduled or promised.

Nothing in this document claims a staging or production deployment, a formal
acceptance, or a performance characteristic.

---

## A. Currently implemented

Each row states the evidence that supports it. Evidence is described by **behaviour**
rather than by internal component or function names.

### A.1 Counter and basket

| Capability | Evidence |
| --- | --- |
| Barcode-scanner input that keeps focus after each accepted scan, so a scanner types straight into the till | The barcode input control refocuses itself after a lookup, after an error and on mount |
| Item lookup by barcode, then by item code, then by item name, with a partial name match offered as a list when an exact match is not found | The lookup path tries an exact barcode match, an exact item-code match and an exact name match in turn, then offers a partial name match of the catalogue |
| Live basket with quantity, unit price, line total and per-line tax | The basket model computes line totals and tax per line |
| Three line kinds in one basket: **sale**, **rental** (daily rate × days) and a **security deposit** held as a liability | The basket line-kind type admits exactly these three |
| Basket summary with **taxable value, CGST, SGST, total tax and grand total**, and a count of chargeable units | The totals panel renders exactly these figures |
| 80 mm thermal receipt layout with a print-specific stylesheet | The receipt view and its print stylesheet, which set the page size to 80 mm |

There is **no discount, markdown or coupon capability** in this release. A basket line
carries a quantity, a unit price, a tax rate, and — for rental goods — a duration in
days; there is no field for a percentage or absolute reduction, and no discount
computation anywhere in the basket or totals logic.

### A.2 Tax

| Capability | Evidence |
| --- | --- |
| Intra-state split of every taxable line into **CGST + SGST** | The tax module splits a taxable amount into its two halves |
| **Paisa-exact** arithmetic: one half is rounded first and the other takes the remainder, so the two always re-add to the tax total | The split function rounds one half first and derives the other from the total |
| Accumulation across lines without rounding drift | The accumulator rounds each component to two decimals as it adds |
| Tax rates modelled as a **closed type of permitted values** | The tax-rate type admits a fixed set of values |

**Two qualifications on the tax-rate type.**

1. The permitted set includes one low rate that exists to serve the **security-deposit
   line**, which is a liability rather than a sale. It is **not** a standard GST slab.
   The rate applied to deposit lines is a commercial and tax matter for the deploying
   organisation and their tax adviser, and is **not published here**.
2. The type is a **compile-time** constraint only. A tax rate read from the back office's
   own tax-template records is accepted and carried through to the receipt **without
   being checked against the permitted set** — the value is cast to the type rather than
   validated against it. In practice the receipt therefore shows whatever rate the back
   office holds for that item, including a rate outside the declared set. The tax
   arithmetic itself imposes no restriction: it accepts any numeric rate.

### A.3 Selling without a connection

| Capability | Evidence |
| --- | --- |
| Local bill store in the browser, schema-versioned | The local store declares a versioned schema with two collections |
| Every completed bill is persisted locally and marked **pending** before any network call | The checkout path writes the bill to the local store before any back-office call |
| A background worker drains pending bills when the connection returns | The worker listens for connectivity events and also polls on a timer |
| A live counter of bills awaiting reconciliation, surfaced in the header | The worker publishes a snapshot including a pending count, which the header displays |
| Online and offline bills are distinguished in the store | A dedicated field records which path produced the bill |

### A.4 Reconciliation and de-duplication

| Capability | Evidence |
| --- | --- |
| Every bill carries a **unique idempotency key** — a one-off reference number | The bill model carries a generated key, and the local store indexes it |
| Before posting, the till checks whether an invoice already exists for that key | The integration layer queries the back office's invoice list for that key |
| Only then is the invoice created | The create call is made only when the query returns nothing |
| The key is stamped onto the back-office invoice, so the two can be correlated later | The setup procedure creates three additional sales-invoice fields, one of which carries the key |
| The key is also sent as a tracing header on the create call | The create request carries the key in a header |

This combination — *check first, then create, keyed* — is what makes a retried bill
reconcile against the same invoice rather than a second one.

**Its limits are stated in full in [`offline-billing.md`](offline-billing.md)** §3: the
check and the create are two separate operations rather than one atomic step, and the
guarantee depends on there being a single writer at a time.

### A.5 Back-office integration

| Capability | Evidence |
| --- | --- |
| Authenticate to the back office as part of a supervisor authorisation | The integration layer performs a sign-in against the back office |
| Import of the product catalogue with name, item code, barcode, selling rate and tax rate | The import reads the item list, the price list for the configured price list, the barcode records, and the tax-template records |
| Import of prices, barcodes and tax templates | As above — prices and tax rates are read from their own back-office records rather than derived locally |
| Push of a bill as a back-office sales invoice | The integration layer creates a sales invoice from a bill |
| Items and taxes are pulled from the back office rather than being re-keyed | The basket is priced from imported master data |

**Limits of the import.** The import reads a **single page** of results — up to 500
items, and up to 2,000 price, barcode and tax-template records — and does **not**
paginate. A catalogue larger than those limits is therefore **silently truncated at
import**, and the till will not hold the remaining items. **Not yet established** — the
expected catalogue size for a single store, and the intended pagination behaviour, have
not been established.

The import also does **not** bring stock levels. The stock quantity recorded against an
imported item is a placeholder rather than a real figure, and the till does not
maintain a stock count of its own.

There is **no cashier sign-in screen and no sign-out control** in this release. The
sign-in operation exists in the integration layer and is exercised as part of a
supervisor authorisation. **Not yet established** — how a till session is established, and
which account bills are posted under, has not been documented for this project.

### A.6 Supervised exceptions

| Capability | Evidence |
| --- | --- |
| Certain till operations cannot be completed by a cashier alone | The basket marks certain lines as requiring authorisation before they can be applied |
| The authorising user must present **credentials that are verified against the back office at the moment of the request** | The authorisation check signs the presenter in against the back office and reads their account roles |
| The **live authorisation** is held in application memory and expires automatically | The grant list is held in memory, filtered by age, and discarded when the bill is closed |
| The **authorisation record** — who approved, when, which line and amount — is captured on the bill and **stored with it** in the local store | The bill model carries an authorisation record, and the bill is written to the local store with it |

The distinction matters: the **live authorisation** is transient and exists only while
the bill is open, whereas the **record** of the authorisation is persisted alongside the
bill in the browser's local store. See [`security-and-privacy.md`](security-and-privacy.md)
§2 and §5.

The specific operations that require authorisation are a governance matter for the
deploying organisation and are not enumerated in this document.

### A.7 Configuration integrity

| Capability | Evidence |
| --- | --- |
| Store identity — name, address, tax registration number, phone — is supplied as build configuration, never hard-coded | The configuration module reads the identity from the build environment only |
| A production build that is missing the store identity **refuses to start** rather than printing a blank or placeholder tax registration number on a receipt | The entry point validates the identity and renders a blocking configuration-error screen instead of the till |
| The tax registration number is format-checked, and placeholder values are rejected | The validation module checks the format and treats generic development placeholders as "not configured" |
| Store identity is validated at start-up, not at first use | The check runs at application entry |

**Scope of the format check.** The check establishes the *shape* of a registration
number. It does **not** verify the checksum digit, and it does not establish validity
with any tax authority. It is a configuration-integrity control, not a tax-authority
integration.

### A.8 The sample catalogue

When the local item store is empty, the application loads a **small sample catalogue**
of twelve sporting-goods lines into it, so that the till screen is usable before any
items have been imported from the back office. The barcodes in that catalogue are
EAN-13-shaped values that a scanner will read successfully.

This loading is **not** restricted to development builds: it is not covered by the
production guard described in §A.7, so a production build with an empty local store will
also load the sample catalogue, and those lines would be sellable until an import
replaces them. Whether the sample catalogue should be excluded from production builds is
a decision that has not been taken.

---

## B. Implemented but not independently verified in a live environment

This section exists because it is easy to present "implemented" as "working". These
capabilities are real in the code, but the **back-office** side has never been
exercised against a live ERPNext instance, and the till's own billing logic is not
covered by an automated test suite — the automated suites cover the deployment tooling
only.

| Capability | What is unverified |
| --- | --- |
| Container stack start-up and the creation of the back-office site | The compose stack and its setup scripts exist; no stack has been brought up end-to-end here |
| Installation of the GST application | The installer script and its branch selection exist; the install has not been executed |
| Creation of the supervisory authorisation account and its user | The setup script exists; it has not been executed against a live back office |
| Creation of the three additional sales-invoice fields | The script exists and is idempotent by construction; it has not been executed |
| A printed receipt on physical 80 mm paper | The layout and print stylesheet exist; no hardware print test has been performed |
| A bill pushed to the back office and appearing as an invoice | The code and the de-duplication contract are implemented; the round trip against a live back office has not been performed, and the billing logic is not covered by an automated test suite |
| An item import against a live back office | The import code exists; it has not been run against a live instance, and its single-page behaviour has not been exercised at scale |

---

## C. Potential future improvements

**None of the following is implemented.** These are ideas for future development — not
built, not scheduled, and not a commitment. They are recorded so the boundary of the
system is explicit, and the fuller picture is in [`roadmap.md`](roadmap.md).

| Direction | Current position |
| --- | --- |
| Frontend component-level automated tests | The till currently has no component test framework; see the "deliberately absent" table in the stack document |
| Payment settlement integration | The till records a payment mode; reconciliation is a back-office activity today |
| Shift / day-end close | Deliberately excluded in phase one |
| Returns and refunds at the till | Back-office today |
| Customer accounts and stored value | Out of phase-one scope |
| Inter-state (IGST) supply | Requires tax handling the current implementation deliberately omits |
| Multi-store / multi-warehouse | Out of phase-one scope |
| Paginated item import | The import reads a single page today; see §A.5 |
| Excluding the sample catalogue from production builds | Not restricted today; see §A.8 |
| Stock tracking at the till | Stock quantities are not imported from the back office, and the till holds no count of its own |
| Cloud hosting, multiple stores, centralised administration | Not designed; see [`roadmap.md`](roadmap.md) §5 |

---

## D. Explicitly not part of the system

Repeated from the overview because it is the most common source of misunderstanding:
customer accounts, loyalty and stored value; purchase and stock-transfer workflows;
returns and refunds at the till; **shift management, cash counting, day-end close and
cash reconciliation**; electronic invoicing; A4 and statutory invoice printing;
multi-store consolidation; customer-facing web or mobile channels; payment settlement
integration; **discounts, markdowns and coupons**; till-side stock tracking.

---

## E. Requirements traceability

**Not yet established** — the accepted requirements baseline, the first-phase scope
statement and the acceptance criteria are not part of this documentation set. The lists
above are derived from the implementation, not from a signed specification, and have
not been reconciled against one. Reconciling them is the precondition for calling this
document a requirements traceability matrix.
