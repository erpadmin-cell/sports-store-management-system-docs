# NOTICE

This notice accompanies the **public documentation** for the Sports Store Management
System.

---

## 1. What this repository contains — and what it does not

This repository contains **documentation only**. It does **not** contain:

- any application source code (neither the counter front end nor the back office);
- any configuration template, container definition or environment file;
- any credential, secret, key or certificate;
- any business identifier, tax registration number, customer record or contact detail;
- any Git history from the private development repository.

The application is **closed-source**. It is developed, reviewed, operated and versioned
separately. Nothing in this repository grants a licence to the application source, and no
part of the application is redistributed here.

**Publishing this documentation does not open-source the application.** The application
source, its configuration and its operational material remain private.

---

## 2. Documentation licence

**No licence file is published with this documentation set.** The terms under which
these documents may be reused have not been decided, and are handled as a separate
internal publication matter.

Until that decision is made and stated, the position is simply that **no permission to
reuse these documents is granted**. The application source remains closed-source and
unavailable regardless.

No licence has been chosen, implied or invented by the publication of this notice.

---

## 3. Third-party components

The system described in these documents is built on the following third-party software.
This section is an **attribution summary** — it deliberately **does not reproduce any
third-party licence text**, and it does not state that any component is used in
compliance with its licence. Reproducing licence texts, preparing the corresponding
attribution files, and confirming the applicable licence position are separate legal
steps that have not been completed.

### 3.1 Application and container components

| Component | Role | Licence family as published upstream |
| --- | --- | --- |
| Frappe Framework | Web framework underpinning ERPNext | MIT |
| ERPNext | Back-office business management system (stock, invoicing, product masters) | GNU General Public License, version 3 (GPL-3) |
| India Compliance app for Frappe/ERPNext | Indian GST: item tax templates, GST invoice behaviour | GNU General Public License, version 3 (GPL-3) |
| `frappe/frappe_docker` | The container-orchestration layout this project follows | MIT |
| MariaDB | Relational database | GNU General Public License, version 2 (GPL-2) |
| Redis | Cache and background-queue broker | **Unconfirmed — see §3.4** |
| nginx (as shipped in the application image) | HTTP server and reverse proxy | BSD 2-Clause |
| Node.js (as shipped in the application image) | Realtime websocket service | MIT |

### 3.2 Two different questions, kept separate

The requirements in the rest of this section fall into **two categories that must not be
confused**, because they have different answers, different owners and different
consequences:

| | **Publishing these documents** | **Distributing the software or the container images** |
| --- | --- | --- |
| **What is involved** | Publishing prose that *describes* the system | Delivering the application source, the built images, or the custom image described in §3.5 |
| **Does it involve the third-party software itself?** | **No.** No third-party source code, image or licence text is reproduced here | **Yes.** The images contain or build upon GPL-3 licensed components |
| **What is required** | The attribution file described in §3.3, and a decision on the documentation licence under §2 | The Redis version and licence position in §3.4, the custom-image question in §3.5, and the compatibility review in §3.3 |
| **Do the §3.4 and §3.5 questions block publishing the documentation?** | **No.** They are open questions about software distribution. This repository distributes no software | **Yes.** They must be answered before any software or image is distributed |

**Neither §3.4 nor §3.5 is a precondition for publishing this documentation.** They are
recorded here because leaving them out of an attribution notice would be misleading, and
because they must be settled before the software itself is handed over.

### 3.3 Attribution obligations

**A. For publishing this documentation**

- A **Third-Party Notices** file listing every component with its exact version and
  licence identifier, together with the required attribution texts, must be produced and
  legally checked before the documentation set is published. The table in §3.1 is a
  summary only, and the Redis entry is deliberately marked incomplete.
- No third-party licence text is reproduced in this repository, and no third-party
  source code or image is redistributed here.

**B. For distributing the software or the images**

- Any **custom Frappe application** built on top of the GPL-3 components for a
  deployment of this system must be reviewed for licence compatibility **before** it is
  distributed to a third party. That review has not been recorded.
- The Redis version and licence position in §3.4 must be settled, and the custom-image
  question in §3.5 must be answered, before any image is delivered.

### 3.4 Redis — applicable licence not yet determined

**The licence that applies to the Redis component has not been confirmed**, and this
document makes no claim about it.

The reason is that the upstream Redis licence has changed across releases, and the
terms differ materially between them. Rather than restate those terms here — which
would risk publishing an inaccurate summary — the position is recorded as
**unconfirmed**:

| Item | Position |
| --- | --- |
| The image this project selects | A Redis image tagged in the `8.6` series, declared in the deployment configuration as a version selection rather than a fixed digest |
| The licence option that applies | **Not determined.** The upstream terms differ between release series, and the option taken depends on the version actually deployed and on how the component is used and distributed. |
| What is required before the software or an image is distributed | The exact version and digest of the deployed image must be confirmed, and the applicable licence option must be selected and recorded. |
| What is required before the documentation is published | Nothing further. The entry above is an honest statement of an open question, and the attribution file in §3.3 records the component and its role. |
| Who must determine this | A qualified legal adviser. |

The same applies to any other component whose licence terms vary by version or
distribution mode.

### 3.5 The custom application image — distribution position pending legal review

**This section records an open question about software distribution. It is not a
conclusion, and it must not be read as one.** It does not bear on publishing these
documents.

The back office can be assembled in two ways:

1. **From the published upstream image**, with the GST application installed into the
   running back office by a setup script; or
2. **As a custom image built by this project**, in which the framework, the ERPNext
   application and the GST application are combined into a single image artefact.

The second route produces an artefact that **incorporates GPL-3 licensed components**
rather than merely consuming them. What follows from that — in particular whether
building and delivering that artefact constitutes distribution, and what source-offer or
other obligation would then arise — is **not determined here**.

| Item | Position |
| --- | --- |
| Whether the custom image is the intended production route | **Not decided for this project.** The deployment documentation describes both routes and does not select one. |
| Whether delivering the custom image constitutes distribution | **Not determined.** Requires legal review. |
| Whether a source offer or other obligation is triggered | **Not determined.** Requires legal review. |
| What is required before the image is delivered | A written legal position on the custom-image route, and confirmation of which route is operated. |
| What is required before the documentation is published | Nothing further. This repository does not contain, build or distribute that image. |

### 3.6 A future hosted offering

If the system were ever offered as a hosted or subscription service, that would raise a
**separate set of questions again**, distinct from both publishing these documents and
distributing the software: multi-store tax registration, payment settlement and data
residency; isolation between tenants; and the commercial, hosting and contractual terms
of the offering.

Should that transition ever be taken, **additional technical documentation, software
licensing reviews, security assessments and compliance work** would be considered as
part of it.

**None of that is settled, and none of it is described as though it were.** It is
recorded in [`docs/roadmap.md`](docs/roadmap.md) §5 as a possible direction. Any such
offering would require its own review, separately from anything in this notice.

---

## 4. Accuracy and scope of these documents

The documents in this repository describe the system as implemented, and **label the
verification status of each claim explicitly**. Claims that could not be verified — in
particular anything about deployment, performance, scale, reliability or operational
acceptance — are either absent or explicitly labelled.

Where a statement in these documents is marked as not yet decided, or where a document
states that something has **not** been verified, that is the case, and it should not be
quoted as a settled position.

---

## 5. Trademarks and naming

This documentation refers to the system by a **descriptive name only**. No brand,
trademark or product name is asserted, and no endorsement of any product or organisation
is implied by the mention of third-party components. The naming and clearance position
for the project is handled as a separate internal matter and is not covered here.

---

## 6. Contact

A contact address for correctness issues, security disclosures and licensing enquiries
is not yet designated, and is a separate internal publication decision.
