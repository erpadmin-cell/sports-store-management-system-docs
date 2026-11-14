# Selling without a connection

> **In plain terms.** This is the capability the whole system is built around, so it
> is worth explaining properly.
>
> **"Offline-first" means the till can complete and print a sale with no network
> connection at all.** The bill is written to the till's own storage first, the receipt
> prints immediately, and the bill is sent to the back office later, when the
> connection returns.
>
> This document explains that model, the protection it gives against recording the same
> bill twice, and — equally important — the limits of that protection.

---

## 1. Why this matters

Most systems that lose their connection fail in one of two ways:

- they **refuse the transaction**, which means the till stops selling; or
- they **lose the transaction**, which means the sale happened at the counter but never
  reached the records.

Neither is acceptable at a counter. The second is worse, because the shop does not know
it has sold something.

This system takes a third position: the sale is **committed locally first** and
reconciled later, and the reconciliation step is designed so that repeating it is safe.

---

## 2. The model

### 2.1 The local bill store

Completed bills are written to a small database built into the browser (IndexedDB,
accessed through Dexie). The store is schema-versioned and holds two collections:

| Collection | Key | Purpose |
| --- | --- | --- |
| Items | Item code | The cached catalogue — name, barcode, selling rate, tax rate — used to price the basket without the back office |
| Bills | Bill identifier | The queue of bills, each with its sync state, its timestamp and its **idempotency key** |

The bill collection is indexed by **bill identifier**, by **sync state**, by
**timestamp**, by the **idempotency key**, and by the composite **state-and-timestamp**
pair that the reconciliation query uses.

The item collection holds whatever catalogue has been imported. When it is empty, a
**small sample catalogue** is loaded into it — see [`features.md`](features.md) §A.8 for
what that means for a production build.

### 2.2 Two states, and one commit point

A bill is either **pending** or **synced**. The transition is deliberate:

```
  sale completed
        │
        ▼
   written locally  ◄── the cashier's commit point
   state = pending
        │
        │  (reconciliation process, when connectivity allows)
        ▼
   posted to the back office
   state = synced
```

**The local write happens before any network call.** Once the bill is in the local store,
the cashier has a completed sale with a printed receipt, regardless of what the network
does afterwards. The network is an optimisation in this design, not a dependency.

Note that a bill is **not removed** from the local store once it has been reconciled. The
browser copy remains, so the counter machine retains a second copy of every bill it has
taken. See [`security-and-privacy.md`](security-and-privacy.md) §2 and §7.

---

## 3. Protection against duplicates

The hard problem here is the **retry**: a bill may be sent, the connection may drop
before the response arrives, and the till cannot tell whether the bill was created. A
naive retry creates a duplicate invoice.

**What an "idempotency key" is.** A one-off reference number generated for each bill.
The same bill always produces the same key, which is what lets the back office
recognise a repeat.

### 3.1 How it works

1. When the bill is created locally, the till generates a **unique idempotency key** and
   stores it on the bill.
2. Before creating anything in the back office, the till **asks the back office whether
   an invoice already exists for that key**.
3. Only if none exists is the invoice created, with the key stamped onto it.
4. The key is also sent as a tracing header on the create call, so that the two sides
   carry the same identifier if a failure is ever investigated.

### 3.2 What this provides, and what it does not

**Provided:** a bill that reaches the back office is reconciled against **one** invoice,
however many times the reconciliation loop retries it, and however the connection
failed.

**The assumptions and limits behind that statement:**

| Limit | Detail |
| --- | --- |
| **The check and the create are separate operations** | The de-duplication sequence is a query followed by a create, not one atomic step. There is a window in which two callers could both find nothing and both create. |
| **No uniqueness constraint at the database level** | The back office stores the key on the invoice as an ordinary data field. It is **not** a constrained or unique index, so the database will not itself reject a second invoice carrying the same key. |
| **A single-writer assumption** | The guarantee holds while there is exactly one writer posting bills to the back office — which is the case for a single counter till, and is the design this project assumes. A second till writing to the same back office would break it. |
| **The back office does not natively honour the tracing header** | The header is sent for correlation; the de-duplication property comes from the query in step 2, not from the header. Whether the back office records that header in its logs has not been established. |

**Not independently verified** — the circumstances under which a duplicate could still
arise have not been formally analysed or tested, and the round trip has never been
exercised against a live back office. Pinning the key with a database uniqueness
constraint, or moving the de-duplication to an atomic server-side operation, would close
the window; neither has been implemented or evaluated.

### 3.3 What is not guaranteed

That a bill is reconciled **at all**, promptly. A bill can remain pending indefinitely if
the back office is unreachable, if the item it references no longer exists in the
catalogue, or if the bill fails validation. Pending bills are visible to the cashier
through the sync indicator, which is the operator's signal that something needs
attention.

---

## 4. The reconciliation process

A single background process owns reconciliation. It:

- listens for the browser's `online` event, so a restored connection triggers a drain
  immediately rather than waiting for the next tick;
- listens for the `offline` event, so the interface reflects the state at once;
- polls on a timer as a safety net, covering cases where the connection returns without
  the event firing;
- drains the pending queue **in order**, publishing a running count so the interface can
  show how many bills are outstanding.

The process is deliberately conservative: it drains rather than floods, so a long outage
does not produce a burst of concurrent requests when connectivity returns. It also stops
draining when it meets a failure it expects to be transient, and continues past a
failure it considers permanent, so that one unfixable bill cannot wedge the queue.

---

## 5. What happens on a conflicting bill

A bill can fail to post for a handful of reasons — the item was deactivated, its tax
template changed, the price no longer matches, or validation failed. The bill stays
**pending** and the process reports the error state. The bill is **not** silently
discarded: it remains in the local store until a person resolves it.

The operational policy for unresolved bills — retry interval, maximum pending age,
manual intervention procedure and who is responsible for it — is a matter for the
deploying organisation. The implementation does not currently expire or escalate a
stuck bill; it remains pending and visible.

---

## 6. The counter machine holds the only copy until reconciliation

A consequence worth stating plainly: **until a bill has been reconciled, the local
store on the counter machine is the only copy of it**. It is not encrypted at rest
beyond what the operating system and browser profile provide, it is not backed up by the
application, and it is lost if the browser profile is cleared or the machine is rebuilt.
After reconciliation the browser copy is retained rather than deleted.

**Neither the cash in the drawer nor an unreconciled bill is reconciled by the till.**
Cash counting, drawer reconciliation, shift management and day-end close are **out of
scope** for this release — the till records transactions and a payment mode, and nothing
more. See [`overview.md`](overview.md) §6 and [`features.md`](features.md) §D.

The accepted risk position and the compensating controls for local bill data (disk
encryption, browser-profile protection, workstation backup, retention) and for cash
accountability are matters for the deploying organisation and are not stated here.

---

## 7. Limitations

| Limitation | Consequence |
| --- | --- |
| One browser profile holds the queue | A different browser or a cleared profile loses pending bills |
| The queue is bounded by browser storage | A very long outage can exhaust the quota; no compaction or queue limit is implemented |
| Reconciled bills are not deleted locally | The counter machine retains a copy of every bill it has taken |
| Reconciliation is serial | A large backlog drains one bill at a time |
| Catalogue and price data are cached | A price change made in the back office is not seen by the till until the next refresh |
| The import reads a single page of results | A catalogue beyond the import limits is silently truncated — see [`features.md`](features.md) §A.5 |
| Tax is computed on the client | If the back office's tax template disagrees with the client's computation, the two will differ — the back office remains authoritative for the invoice |
| No local reprint queue | A bill lost from the browser store cannot be reprinted from the till |
| De-duplication assumes a single writer | A second till against one back office is outside what the guarantee covers |

---

## 8. Testing

The reconciliation logic and the de-duplication contract are **not** covered by an
automated test suite. The automated suites described in
[`technology-stack.md`](technology-stack.md) cover the deployment credential
safeguards, the command classification and the environment-file parsing; they run
entirely in a sandbox with synthetic data and never contact a back office, and they do
not exercise the till's own billing logic.

**Not independently verified** — the following have **not** been carried out and are
therefore not claimed:

- interruption testing with a real back office under a degraded network;
- soak testing of a long outage followed by a full drain;
- browser-storage-exhaustion testing;
- testing of the de-duplication behaviour under concurrent posting;
- a physical till test with a real receipt printer and a real scanner.
