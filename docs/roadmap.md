# Future possibilities

> **In plain terms.** This document is **not** a plan, a schedule, or a list of
> commitments. It records:
>
> 1. where the current implementation deliberately stops, so the boundary is explicit;
> 2. the work that would need to happen before the system could be called validated;
> 3. the directions the project may want to move in.
>
> Nothing in sections 3 or 4 is built, scheduled or promised. The current system is a
> **single-store installation on the retailer's own hardware**, and that remains the
> system the rest of these documents describe.

---

## 1. What this document is, and is not

| This document is | This document is not |
| --- | --- |
| A description of where the implementation stops today | A schedule, a release plan or a set of dates |
| A list of boundaries that may be lifted in future | A promise that any of them will be lifted |
| A record of work that has **not** started | A statement of intent, commitment or delivery |
| A record of the direction the project has indicated | An agreed or funded plan |

Terms used throughout:

| Term | Meaning |
| --- | --- |
| **Implemented** | Present in the code, with the supporting evidence cited |
| **Implemented, unverified** | Present in the code, never exercised end-to-end |
| **Candidate** | A possible direction. Not agreed, not scheduled, not started |

---

## 2. The current position, in one paragraph

The first phase has **implemented** a single-store, single-state point of sale with
offline-first billing, intra-state GST, an 80 mm thermal receipt and supervised
exceptions.

What has been **verified** is narrower than what has been implemented: the application
type-checks and builds cleanly, and the automated suites covering the deployment
tooling pass. What has **not** happened is any end-to-end exercise — the ERPNext
provisioning path, the invoice round trip and a physical receipt print have never been
run against a live back office, and the system has **not** been deployed, **not** been
formally accepted, and **not** been shown to complete a sale on a counter. A successful
build is not the same thing as a validated system.

The most valuable next step is therefore not a new feature; it is to complete the
verification of what already exists. See [`features.md`](features.md) §B for the full
list.

---

## 3. Work that would need to happen before the system could be called validated

These are the gaps that stand between the current implementation and a system whose
behaviour is known. None of them is a feature.

| Candidate | What it would establish |
| --- | --- |
| **Provision and exercise the back office** | That the container stack starts, the site is created, the GST application installs and the additional sales-invoice fields are created — as a repeatable, documented procedure |
| **Invoice round trip against a live back office** | That a bill pushed from the till appears as the expected invoice, with the tax split and the bill identifier matching |
| **Interrupted-connection test** | That a bill transmitted on a link that drops mid-flight reconciles against one invoice rather than two |
| **De-duplication under concurrent posting** | Whether the check-then-create sequence behaves as the single-writer assumption requires — the limitation set out in [`offline-billing.md`](offline-billing.md) §3 |
| **Long-outage and full-drain test** | That the pending queue drains completely and correctly after an extended disconnection |
| **Physical till test** | That the receipt renders and prints correctly on 80 mm paper, and that the scanner input behaves as designed on the target hardware |
| **Browser storage-exhaustion test** | The point at which the local queue fails, and what the operator sees when it does |
| **Import at realistic catalogue size** | Whether the single-page import truncates a real catalogue, and what the cashier sees when it does — see [`features.md`](features.md) §A.5 |

The verification plan, the environment it runs in, and the acceptance criteria that
would close each item are decisions for the deploying organisation.

---

## 4. Possible directions

Each of these is a boundary the first phase deliberately set. Whether the boundary is
lifted is a decision for the deploying organisation.

| Direction | Current position |
| --- | --- |
| **Shift, cash count and day-end close** | Out of scope in phase one. The till records transactions and a payment mode; it does not count a drawer, manage a shift, close a session, or reconcile cash. |
| **Returns and refunds at the till** | Handled in the back office today. Bringing them to the till would require a negative-value transaction path and a decision on how a refund interacts with a reconciled invoice. |
| **Payment settlement** | The payment mode is already a first-class field on a bill, so the data model supports this; settlement itself is a back-office activity. |
| **Customer accounts and stored value** | Out of phase-one scope. The system captures no customer identity today, and adding it would be a material change to the data-handling position described in [`security-and-privacy.md`](security-and-privacy.md). |
| **Inter-state supply** | Deliberately excluded. The tax model implements an intra-state split only; adding inter-state supply would require a materially different tax path, and would change what appears on the receipt. |
| **Multiple stores or warehouses** | Out of phase-one scope. The current configuration model assumes a single store in a single tax jurisdiction. A second writer would also break the de-duplication assumption described in [`offline-billing.md`](offline-billing.md) §3. |
| **A customer-facing channel** | The back office already exposes the product catalogue over its REST interface, but a public catalogue needs its own read model and its own pricing rules. |
| **Discounts, markdowns and coupons** | Not present in this release. The bill model has no field for a reduction, so this would be a change to the data model rather than an addition to it. |
| **Till-side stock tracking** | Not present. Stock quantities are not imported from the back office, and the till holds no count of its own. |

---

## 5. Cloud hosting, multiple stores and a subscription service

**This is the direction the project has indicated, not a plan, and not a commitment.**
Nothing in this section is implemented, and no design, schedule, hosting model or
commercial arrangement has been agreed.

| Direction | Current position and what it would require |
| --- | --- |
| **Hosted, cloud-based back office** | The current deployment is a container stack operated on the retailer's own hardware. A hosted model would move the back office to a managed environment, and would require the network, encryption, backup, monitoring and access-control arrangements that are currently undecided — see [`deployment-overview.md`](deployment-overview.md). |
| **Multiple stores on one installation** | The current configuration model carries a single store identity compiled into the till bundle, and the de-duplication contract assumes a single writer. Multiple stores would require per-store configuration, per-store tax jurisdiction, and a tenancy-aware de-duplication mechanism. |
| **Centralised administration** | The till bundle is currently built and deployed as a static artefact with no delivery mechanism defined. A hosted model would need a defined delivery, versioning and rollback path. |
| **Tenant isolation and per-tenant configuration** | The store identity is currently compiled into the bundle rather than held per tenant, and the back office runs as a single installation. Serving several stores from one installation would change both the configuration model and the data-handling position described in [`security-and-privacy.md`](security-and-privacy.md). |
| **Hosted availability, backup and recovery** | The current design is a single stack with no high-availability, disaster-recovery, backup or restore procedure defined. |
| **Identity and access management** | The current model relies on back-office accounts. A hosted platform would need an access model appropriate to it. |
| **A subscription service** | Turning the product into a service the retailer pays for would raise its own commercial, hosting, contractual and data-handling questions. |
| **Hosted multi-store tax and settlement compliance** | A platform spanning stores and jurisdictions would raise tax-registration, settlement and data-residency questions that do not arise for one store in one region. These would need their own review, separately from the product design. |
| **Review work that a hosted offering would call for** | Should the project ever transition to a subscription service, additional technical documentation, software licensing reviews, security assessments and compliance work would be considered as part of that transition. None of that work has been carried out, and none of it is a commitment. |

The current single-store, on-premises installation remains the system described by the
rest of these documents.

---

## 6. Engineering candidates

| Candidate | Current position |
| --- | --- |
| **Front-end component tests** | The till application has no component-test framework today. Its front-end correctness rests on static typing alone; the automated suites cover the deployment tooling only. Whether a front-end test framework is introduced has **not** been agreed or scheduled. |
| **Queue compaction and a queue limit** | The pending queue is bounded only by the browser's storage quota. No limit, no compaction and no maximum-pending-age policy is implemented. |
| **A local reprint path** | A bill that has been lost from the local store cannot be reprinted from the till today. |
| **Automatic escalation of a stuck bill** | A bill that cannot be posted stays pending and visible to the operator. Nothing escalates it or ages it out. |
| **Local retention policy** | Reconciled bills are never removed from the browser's local store, and no retention or compaction policy exists. |
| **Observability** | Logging, metrics and alerting are not part of the current implementation. The operator's signal is the pending-bill indicator in the till header. |
| **Immutable version selection** | The framework, ERPNext and GST applications are selected by moving branch rather than by commit, so a deployment is not strictly reproducible at a later date. Pinning them to commits would close that. |
| **Paginated import** | The item import reads a single page of results and silently truncates beyond its limits. |
| **Restricting the sample catalogue to development** | The sample catalogue is loaded in any build, including production. |
| **Enforcing the tax-rate type at runtime** | A rate read from the back office is cast to the declared type rather than checked against it, so an out-of-set rate can reach the receipt. |
| **Real stock quantities at the till** | Imported stock is a placeholder; the till holds no count of its own. |

---

## 7. Operational work that has not been started

The following are operational activities that the system assumes but that do not
exist yet. They are listed because they affect how the system would be run, not
because they are features of it.

| Item | Status |
| --- | --- |
| Deployment target platform and hosting arrangement | Not finalised |
| Image registry and the promotion path between environments | Not defined |
| Backup and restore procedure | Not defined |
| Monitoring, alerting and log aggregation | Not defined |
| Runbook: day-to-day operation, escalation and incident procedure | Not written |
| Rollback and change-approval process | Not defined |
| High-availability or disaster-recovery design | Not designed; the current model is a single stack |
| Data retention and archival policy | Not defined |
| Encryption at the web endpoint and certificate handling | Not configured in the implementation; not yet decided |
| Cash-handling and drawer-reconciliation procedure | Not defined; out of scope in the application |
| Till bundle delivery and update mechanism | Not defined |
| Provisioning of a fresh counter machine | Not defined; the sample catalogue is currently the only thing on a new machine |

**Not yet established** — the requirements specification, the acceptance criteria and
the signed business-rule baseline for this project are not part of this documentation
set. The candidate lists above are derived from the implementation and have not been
reconciled against an approved requirements document.

---

## 8. How this document should be maintained

If a plan is agreed, this file should be replaced by it, with each item carrying a
responsible party, a status and a reference to the decision that authorised it. Until
then, the labels in §1 should be preserved by anyone editing this file, so that a reader
cannot mistake a boundary for a plan.

---

## Related documents

- [`features.md`](features.md) — the implemented / unverified / future split.
- [`offline-billing.md`](offline-billing.md) — the verification gaps in §3 in context.
- [`deployment-overview.md`](deployment-overview.md) — the operational work in §7.
- [`security-and-privacy.md`](security-and-privacy.md) — the data-handling constraints
  that a customer-accounts candidate would change.

---

*Documentation for the Sports Store Management System.*
