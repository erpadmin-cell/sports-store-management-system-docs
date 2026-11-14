# Deployment overview

> **In plain terms.** This document describes how the back office is packaged and run.
> It deliberately omits hostnames, network addresses, port assignments, container names
> and credential mechanics — those belong in operational documentation that is not
> published here.

---

## 1. The deployment model

The back office runs as a set of **Linux containers** described by Docker Compose files.
A container here means one of several separate programs — the API, the web server, the
background workers, the scheduler — each in its own isolated environment, all started
and stopped together by one set of commands.

The till application is a **static web bundle** produced by the build. How that bundle is
packaged, delivered to a counter machine and served there is part of a release process
that has not been defined for this project; no delivery or serving mechanism is present
in the current implementation.

Two consequences shape everything else:

- The operator's machine is a **workstation**, not a server. It runs the container
  lifecycle scripts; the data lives in the containers.
- A deployment is **driven from configuration** rather than from code: the operator
  scripts read their settings from a configuration file, and the version selections are
  declared there. **Not yet established** — the deployment is **not** strictly reproducible
  at a later date, because the framework, the ERPNext application and the GST
  application are selected by **moving branch** rather than by an immutable commit. See
  [`technology-stack.md`](technology-stack.md) §2.

---

## 2. Conceptual topology

```
                        operator workstation
                     ┌────────────────────────┐
                     │  till application      │
                     │  (static web bundle)   │
                     │        │               │
                     │        │ REST over     │
                     │        │ HTTP(S)        │
                     └────────┼───────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  web endpoint       │   serves the back-office web
                    │  (web container)    │   interface and routes API calls
                    └─────────┬──────────┘
                              │
              ┌───────────────┼──────────────────┐
              │               │                  │
      ┌───────▼──────┐ ┌──────▼───────┐ ┌────────▼────────┐
      │ application  │ │  realtime    │ │    background   │
      │ (API)        │ │  (websocket) │ │    workers and  │
      │              │ │              │ │    scheduler    │
      └───────┬──────┘ └──────┬───────┘ └────────┬────────┘
              │               │                  │
              └───────────────┼──────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  database           │
                    │  (not exposed to     │
                    │   the host)          │
                    └──────────────────────┘

                    ┌──────────────────────┐
                    │  cache + queue       │
                    │  (not exposed to     │
                    │   the host)          │
                    └──────────────────────┘
```

*Diagram authored from the implementation for documentation purposes. It should be
reviewed, and converted into a branded asset, before publication.*

**Structural facts:**

- The **database and the cache are never published to the host network**. They are
  reachable only from inside the container network, so neither is exposed to the local
  network.
- Only the **web endpoint** is published, and it is the single entry point for both the
  back-office web interface and the API.
- The application is stateless from the container's point of view; all durable state is
  in the database, and background work is driven by the queue.
- The **till bundle is separate from the back-office web interface**. It is a static
  build, and it is not part of the container stack.

The back office is assembled from several cooperating containers: an application
container serving the API, a realtime container for the websocket channel, worker
containers draining the short and long background queues, a scheduler container, a web
container serving the desk assets, and a one-shot configuration container that prepares
the bench on first start. The diagram groups these into the three roles the till
interacts with.

### Transport security

**TLS termination is not configured in the current implementation**, and the production
arrangement is not yet decided. The container configuration defines no certificate
material and no TLS listener, and the till can be pointed at the back office over either
scheme depending on how it is hosted. The production transport arrangement — including
where encryption is terminated, and how certificates are issued and renewed — is a
decision that has not been taken, and the diagram above therefore does not assert it.

---

## 3. Environment separation

The system distinguishes three environments by an explicit declaration in the
configuration file, which the deployment tooling reads before starting anything:

| Environment | Purpose | Credential posture |
| --- | --- | --- |
| **Local** | Developer workstation | May run with the documented placeholder credentials, which the tooling reports clearly when they are in use |
| **Staging** | Pre-production verification | Placeholder credentials are **rejected**; unique, strong values are required |
| **Production** | Live operation | Placeholder credentials are **rejected**; unique, strong values are required |

The declaration must be set **deliberately**. A configuration file generated from the
template leaves it unset, and an unset declaration **prevents the stack from starting**
rather than defaulting to anything. This is deliberate: a silent default is how a
placeholder credential reaches a live system.

The tooling also **rejects duplicate configuration keys** for the sensitive settings, and
**rejects a malformed or whitespace-only credential** — an ambiguity in how a value is
read is treated as a configuration error rather than being resolved silently.

The deployment target platform, the network topology, the encryption arrangement, the
backup and restore procedure, and the monitoring and alerting stack have not been
finalised and are therefore not documented here.

---

## 4. Configuration model

| Category | Examples | Where it lives |
| --- | --- | --- |
| **Secrets** | Database password, back-office administrator password | The environment's configuration file, which is excluded from version control |
| **Environment declaration** | Which environment this is | The same configuration file |
| **Version selections** | Framework branch, application image tag, database and cache images | The same configuration file, and the compose definitions |
| **Endpoints** | Back-office site name, publish settings | The same configuration file |
| **Store identity** | Store name, address, tax registration number, phone | Build-time configuration for the till bundle |

The separation between the last two rows matters: **the store identity is compiled into
the till bundle**, not read from a file at runtime, so it cannot be altered on a live
workstation. A production bundle that lacks a valid store identity **refuses to start**
and shows a configuration-error screen instead of the till.

---

## 5. Release process (as implemented)

1. The till bundle is built and type-checked from a clean checkout.
2. The back-office containers are built or pulled at their selected versions.
3. The site is created, the GST application installed and the additional sales-invoice
   fields added — all by **idempotent** scripts, so any step can be re-run safely.
4. The store identity is supplied as build configuration and the bundle is produced.

The release pipeline, the image registry, the promotion path between environments, the
rollback procedure and the change-approval process have not been defined for this
project.

### Durability of the GST application installation

The GST application can be installed two ways, and they differ in durability:

| Route | Durability |
| --- | --- |
| **Installed into the running back office at setup time** | The application lives in the container's writable layer, while only the sites directory is a mounted volume. The installation therefore **survives a restart but does not survive a container recreation** — a forced recreate, an image switch, or a full stop and start would remove it. |
| **Baked into a custom image at build time** | The installation travels with the image and is therefore durable across recreation. |

The scripts themselves record this distinction and recommend the second route for a
durable installation. Which route is operated has not been decided.

---

## 6. Operating the stack

The lifecycle is driven by a small set of operator scripts — start, stop, create the
site, install the GST application, add the additional sales-invoice fields, provision the
supervisory account, and build the image. They are written for a Windows operator
workstation.

Two operating properties are worth knowing:

- **Teardown never depends on the configuration being correct.** Stopping or removing a
  stack works even if the configuration file is incomplete or damaged, because those
  operations neither create nor start a container. This matters during incidents: you
  can always bring the stack down.
- **Starting always depends on the configuration being correct.** The same tooling
  refuses to start a stack with missing or placeholder credentials, so a misconfigured
  deployment fails immediately and visibly rather than running insecurely.

**Not yet established** — the runbook (day-to-day operating steps, escalation contacts,
incident procedure and recovery drills) has not been written and is not part of the
current deliverable.

---

## 7. What is not documented here, and why

| Omitted | Reason |
| --- | --- |
| Hostnames, addresses, port assignments, container names | Operational detail that belongs in a private runbook and adds nothing for a reader |
| Credential mechanics and the internal safeguards review | Internal security engineering; this document describes only the observable behaviour |
| Backup and restore procedure | Not yet defined |
| Monitoring, alerting and log aggregation | Not yet defined |
| High-availability or disaster-recovery design | Not designed; the current model is a single stack |
| Encryption arrangement and certificate handling | Not configured in the implementation and not yet decided |

---

## 8. Honest statement of deployment status

**No staging or production deployment of this system exists.** The packaging and the
automation described above are implemented, and the parts that can be verified without a
running back office have been; the ERPNext provisioning path has not been executed
end-to-end. Nothing in this document should be read as a deployment claim.
