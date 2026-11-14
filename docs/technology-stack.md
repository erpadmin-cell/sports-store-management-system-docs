# Technology stack

> **In plain terms.** This document explains what each piece of technology is *for*.
> The reasoning matters more than the names: knowing that a system uses React tells you
> little, but knowing that React was chosen because the counter screen is dense with
> many small, independently-updating parts tells you a great deal.
>
> Every component listed here is present in the implementation.

The **Version** column distinguishes three kinds of reference, because they behave
differently and it would be misleading to present them as one:

| Kind | Meaning |
| --- | --- |
| **Resolved** | An exact version fixed by a committed lockfile. Reproducible from the lockfile. |
| **Declared range** | A version *range* declared in the package manifest. The exact version actually installed is fixed by the lockfile, not by the range. |
| **Branch** | A moving upstream branch. The code it resolves to **changes over time**. |

---

## 1. The counter application

These are the libraries the cashier's screen is built from.

| Component | Version | Kind | What it is for, and why it was chosen |
| --- | --- | --- | --- |
| **React** | 19.3.1 | Declared range (`^19.3.1`), resolved by lockfile | A component model that keeps a dense, stateful screen predictable. The till is one screen with many small, independently-updating parts — the basket, the totals, the sync indicator, a modal. React updates only the parts that changed, which keeps those updates cheap on modest counter hardware. |
| **TypeScript** | 7.0.2 | Declared range (`^7.0.2`), resolved by lockfile | The domain is genuinely type-heavy — bills mix sales, rentals and deposits, each with its own tax rate, and a bill moves between "pending" and "posted" states. Static types stop a cashier-facing surface from silently drifting between those states. |
| **Vite** | 8.3.5 | Declared range (`^8.3.5`), resolved by lockfile | The development server and the production bundler. The till must start quickly on a counter machine and produce a small, cacheable bundle. |
| **Tailwind CSS** | 4.3.4 | Declared range (`^4.3.4`), resolved by lockfile | A utility-first styling framework. It gives consistent spacing and typography across a dense interface without maintaining a bespoke stylesheet, and keeps the stylesheet small — which matters when the counter has a slow connection. |
| **Dexie** + **dexie-react-hooks** | 4.4.7 / 4.4.0 | Declared range, resolved by lockfile | A typed, promise-based wrapper over **IndexedDB**, the small database built into the browser. IndexedDB rather than simpler browser storage is what makes the offline store viable: it is transactional, asynchronous, and large enough to hold a working queue of bills. The React bindings let the interface subscribe to live queue counts. |
| **Lucide** | 1.67.0 | Declared range (`^1.67.0`), resolved by lockfile | A single, consistent icon set. Visual consistency on a till matters more to a cashier than icon originality. |

**Not independently verified** — no frame-rate, latency or throughput measurement has
been taken on target counter hardware, and none is claimed.

### Deliberately absent

| Not used | Consequence |
| --- | --- |
| **Router** | The till is a single screen. A router would add a navigation concept a cashier never needs. |
| **Front-end test framework** | The counter application has **no component-test framework**. Its front-end correctness rests on **static typing alone**. The automated suites described in §4 exercise the **deployment tooling** only; they execute none of the application's own logic. Whether a component-test framework is introduced is an open question; it has not been agreed or scheduled. |
| **State-management library** | The till's state is small and local. React's own state and a handful of hooks are sufficient; a global store would be premature. |
| **Internationalisation layer** | The deployment is single-locale. |

---

## 2. The back office

These provide the shared system of record: the product catalogue, the prices, the tax
rules, and the authoritative invoice for every sale.

| Component | Version | Kind | What it is for, and why it was chosen |
| --- | --- | --- | --- |
| **Frappe Framework** | `version-15` | **Branch** | The application framework beneath ERPNext: the REST interface, background workers, scheduling, and the document model. The till consumes it purely over its documented REST interface, which keeps the integration surface small and stable. |
| **ERPNext** | `v15.124.3` (container image tag) | Exact image tag — the applications within it are branch-referenced | The back-office system itself: product master, price lists, barcodes, tax templates, sales invoices and the whole back-office workflow. Reusing an established system rather than building one avoids re-implementing accounting, stock and reporting. |
| **India Compliance app** | `version-15` | **Branch** | The GST module for ERPNext: item tax templates, intra-state invoice behaviour, and the tax structure the till mirrors on screen. It is the reason the receipt can show a CGST + SGST split that matches the back office. |

### How the back office is versioned

The container orchestration this project follows publishes its own branches separately
from the application branches it builds, so version selection is done by combining an
image tag with branch references:

- the **framework** and the **ERPNext application** are selected by branch;
- the **container image** is selected by an exact published tag, so the base image
  itself is fixed;
- the **GST application** is selected by branch, and the setup script that installs it
  **refreshes that branch to its current head on each run**.

**Consequence:** the base image is immutable, but the application layers within it are
not. A rebuild performed at a later date may therefore produce a **different** back
office than one produced today, even though the image tag is unchanged. Pinning the
three application layers to specific commits would remove that variability; that is a
candidate improvement, not a current property of the system.

### The customisations this project adds to the back office

Three additional fields on the sales invoice are created during setup, so the back office
can reconcile what the till produced:

| Field | Purpose |
| --- | --- |
| Offline bill identifier | The bill identifier generated at the till |
| Idempotency key | The unique key the till attaches to a bill, used to avoid reconciling the same bill twice |
| Offline-origin marker | Records whether the invoice was produced through the offline local-first path, rather than a direct online post |

These are created by an idempotent setup script that can be re-run safely.

---

## 3. Data, cache and queue

| Component | Version | Kind | What it is for, and why it was chosen |
| --- | --- | --- | --- |
| **MariaDB** | 11.8 | Exact image tag | The relational database behind the back office — the structured store holding products, prices, invoices and stock. It is the image's own database rather than separately provisioned, so the whole stack can be reproduced from one compose file. |
| **Redis** | 8.6 series | Image tag — see the licence note below | Two distinct roles. As a **cache**, it keeps repeated lookups fast without re-reading the database every time. As a **queue**, it carries the back office's background work — sending e-mail, running scheduled jobs, and processing documents that take a while. |

Neither the database nor the cache is published to the host — they are reachable only
from inside the container network.

> **Licence note.** The Redis component's applicable licence depends on the exact
> version deployed and on how it is used and distributed, and upstream terms differ
> between release series. The applicable option has **not** been determined. See
> [`../NOTICE.md`](../NOTICE.md) §3.4.

---

## 4. Testing and engineering practice

| Practice | What it means |
| --- | --- |
| **Static typing** | The whole counter application is type-checked as part of the build; a type error fails the build. |
| **Build verification** | The production bundle builds cleanly from a clean checkout. |
| **Automated test suite** | **150 assertions** across two sandboxed suites, all passing, zero failures. They cover the **deployment tooling**: the credential safeguards, the command classification, and the environment-file parsing (including malformed and duplicated definitions). **They do not exercise the counter application's billing logic, the tax arithmetic or the reconciliation behaviour.** |
| **Test isolation** | The suites run entirely inside throw-away temporary directories, use synthetic values only, never touch a real environment file, never require a network, and clean up after themselves even when a test fails. |
| **Line-ending discipline** | Repository content is normalised to LF; Windows-only scripts are checked out with CRLF so PowerShell tooling behaves predictably across operating systems. |

---

## 5. Packaging

The back office is packaged as a set of Linux containers described by Docker Compose
files, operated from a Windows workstation by PowerShell scripts that wrap the
container lifecycle. A **container** here means one of several separate programs — the
API, the web server, the workers, the scheduler — each in its own isolated environment,
all started and stopped together by one set of commands. See
[`deployment-overview.md`](deployment-overview.md) for the conceptual topology and the
operating model.

The production hosting platform, the image registry, the network topology, the TLS
arrangement and the build-and-release pipeline have not been finalised, and are
therefore not documented here.

---

## 6. Version summary

| Component | Declared version | Kind |
| --- | --- | --- |
| React / React DOM | 19.3.1 | Range, resolved by lockfile |
| TypeScript | 7.0.2 | Range, resolved by lockfile |
| Vite | 8.3.5 | Range, resolved by lockfile |
| Tailwind CSS | 4.3.4 | Range, resolved by lockfile |
| Dexie / Dexie React Hooks | 4.4.7 / 4.4.0 | Range, resolved by lockfile |
| Lucide React | 1.67.0 | Range, resolved by lockfile |
| Frappe Framework | `version-15` | **Branch** |
| ERPNext (container image) | `v15.124.3` | Exact image tag |
| ERPNext (application) | `version-15` | **Branch** |
| India Compliance app | `version-15` | **Branch** |
| MariaDB | 11.8 | Exact image tag |
| Redis | 8.6 series | Image tag — licence option unconfirmed |

All of the above resolve to published upstream releases or branches at the time of
writing. The **branch** entries are the ones to re-check before a deployment, because
they do not identify an immutable version.
