# CPE Lab Inventory: build spec

A custom web app for the CPE design lab that tracks what the lab owns, where it is, who has it, and what needs ordering. It covers Loans, restocking, Purchase Requests, and label printing.

The decisions here come from the wayfinder map, [uf-cpe-design-lab/stock#1](https://github.com/uf-cpe-design-lab/stock/issues/1). Each section links the ticket that holds the reasoning. Vocabulary follows [`GLOSSARY.md`](../GLOSSARY.md); the build-vs-adopt choice is [ADR 0001](adr/0001-custom-build-over-inventory-tools.md).

Scale: under 500 Catalog Items, a few cabinets, one lab.

---

## 1. People and Permissions

([#11](https://github.com/uf-cpe-design-lab/stock/issues/11), [#14](https://github.com/uf-cpe-design-lab/stock/issues/14))

**Accounts.**
- Self sign-up for anyone with a verified **@ufl.edu** email, signing in by **magic link** (no passwords).
- Everyone starts as a **Lab User**.
- Accounts are never deleted. Staff can deactivate one; deactivating someone who has open Loans shows a warning listing those Loans.

**Model.**
- What a person may do is the union of their **Role**'s Permissions and any individual grants.
- Roles are editable bundles of Permissions.
- Only **Course Staff** Permissions are scoped (to the Courses they're assigned to). Every other Permission applies lab-wide.

**Everyone signed in** can request Loans and submit Purchase Requests. **Raising a Restock Report needs no sign-in.**

| Permission | Lab Manager | Lab Assistant | Course Staff (own Courses) | Lab User |
|---|:-:|:-:|:-:|:-:|
| Approve Loans & extensions | ✓ | ✓ | | |
| Create walk-up Loans / fulfill / check in | ✓ | ✓ | | |
| Approve Purchase Requests | ✓ | ✓ | | |
| Manage the Order List (ordered / received) | ✓ | ✓ | | |
| View spend reports | ✓ | ✓ | | |
| Edit Catalog Items & counts | ✓ | ✓ | | |
| Edit Categories | ✓ | ✓ | | |
| Manage Locations & print labels | ✓ | ✓ | | |
| Run Scan Sessions | ✓ | ✓ | | |
| Manage Restock Needs | ✓ | ✓ | | |
| Manage Projects & Semesters | ✓ | ✓ | ✓ (Projects) | |
| Manage Teams | ✓ | ✓ | ✓ | |
| View a Course's Loans & Purchase Requests | ✓ | ✓ | ✓ | |
| Manage people, Roles & Permissions | ✓ | ✓ | | |
| Edit the Purchase Request terms text | ✓ | ✓ | | |

- Holders of *manage people* grant Permissions, edit Roles, and assign Course Staff to Courses.
- Individual grants are the way to give a student worker a single capability, e.g. *Run Scan Sessions*.

---

## 2. Domain model

| Entity | Key fields | Notes |
|---|---|---|
| **Catalog Item** | name, stock kind (Consumable / Loanable / Equipment), Category, Tags, counted? (Consumables only), default vendor, vendor part number, vendor URL, image | One stock kind and one Category each. |
| **Category** | name, parent | Function-based tree of any depth ([#5](https://github.com/uf-cpe-design-lab/stock/issues/5)). |
| **Tag** | name | Free-form and staff-only, with autocomplete ([#22](https://github.com/uf-cpe-design-lab/stock/issues/22)). |
| **Location** | ID (e.g. `A-07`), name, parent, kind (room / cabinet / shelf / drawer / bin / bench) | Nests to any depth ([#23](https://github.com/uf-cpe-design-lab/stock/issues/23)). |
| **Placement** | Catalog Item, Location, on hand (counted items only) | An item can be in many Locations, and a Location can hold many items. Counts live here. |
| **Unit** | ID (existing lab number, e.g. `SCOPE-01`), Catalog Item (Equipment), home bench Location, status (at bench / on Loan / out of service) | Existing numbers are kept ([#20](https://github.com/uf-cpe-design-lab/stock/issues/20)). |
| **Semester** | name, end date | Kept by Lab Staff. |
| **Course** | code (e.g. `Fall26-Design`), Semester, Course Staff | One per-semester offering. |
| **Project** | name, end date, Course (optional), Team (Lab Users), Cubby | Created by hand each semester. No sheet import. |
| **Loan** | borrower, Catalog Item, quantity (or Unit), Project (optional), status, due date, overdue flag, per-unit outcomes, via Purchase Request? | See §3. |
| **Restock Report** | Catalog Item, level (Low / Out), note, reporter (optional), source (scan / Scan Session) | Grouped into a Restock Need. |
| **Restock Need** | Catalog Item, status, its Restock Reports | At most one open (Open or Ordered) per item. |
| **Purchase Request** | requester, purpose, Project (optional), needed-by (optional), certified-checked, agreed-to-terms, status | Contains Line Items. |
| **Line Item** | quantity, product (vendor search result / URL / free text), vendor, unit price, Catalog Item (linked at receipt), non-preferred reason, status, received quantity | Approved or denied individually. |
| **Preferred Vendor** | name, API capability (search + lookup / link-only), cart-export format | Digi-Key, Mouser, Adafruit (API); Amazon, Grainger (link-only). |
| **Print Job** | label payloads, layout, status (queued / sent) | Picked up by the lab print agent. |

---

## 3. Counting and state rules

([#16](https://github.com/uf-cpe-design-lab/stock/issues/16))

**Counts for a Loanable (or counted Consumable):**

```text
on hand   = Σ Placement.on_hand
out       = Σ outstanding units on Out Loans
reserved  = Σ quantity on Approved Loans
owned     = on hand + out
available = on hand − reserved
```

**Changing the counts:**

| Event | What changes |
|---|---|
| Fulfilling a Loan | Lowers on hand at the Placement staff pick (default: the one with the most on hand). |
| A Returned unit | Goes back into a Placement (default: the one it came from). |
| A Write-off | Leaves `out` and never returns. |
| A Count scan | Sets a Placement's on hand and records an adjustment entry (who, when, old → new). |

**Equipment:** available = Units at bench − reserved. Staff pick the Unit at fulfillment.

**Loan:**

```text
Requested ─▶ Approved (reserves) ─▶ Out ─▶ Returned
    │             │
    ▼             ▼
  Denied      Cancelled (by staff)
```

- **Walk-up Loans** are created straight into **Out** by staff. They can only take available (unreserved) units; to use a reserved unit, staff cancel that reservation first.
- **Fulfilling** means staff put the items in the Project Team's **Cubby**, or leave them out for a non-Project borrower. The Loan is Out from that moment. There's no pickup step and no reservation expiry.
- **Due date** defaults to the Project's end date, otherwise the current Semester's end. Staff can override it.
- **Extensions:** the borrower requests a new date and Lab Staff approve it.
- **Overdue** is a flag on an Out Loan past its due date. While it's set:
  - the borrower can't request new Loans;
  - staff walk-ups for them show a warning but still go through;
  - returning the item clears it.
- **Check-in, per unit:** Returned, Write-off (damaged) or Write-off (lost).
  - Partial check-ins are fine; the Loan stays Out until every unit has an outcome.
  - A damaged Equipment Unit becomes **out of service** at its home bench instead of being written off. Staff restore or move it by hand.
- The **holder** is one person. Any member of the Project's Team can return the items. No borrowing limits.

**Restock Need:**

```text
Open ─▶ Ordered ─▶ Restocked
  └─▶ Dismissed
```

- A new Restock Report joins the item's Open or Ordered Need. After Restocked or Dismissed, the next report opens a fresh Need.
- Marking a counted item Restocked asks how many arrived and which Placement they went into.

**Line Item:**

```text
Submitted ─▶ Approved ─▶ Ordered ─▶ Received
    └─▶ Denied            └─▶ (partial: remainder stays Ordered, or is cancelled)
```

The Purchase Request's own status is derived from its lines: Submitted → Approved / Denied / Needs changes → Ordered → Received → Closed. *Needs changes* sends the request back to the requester to edit and resubmit.

---

## 4. Workflows

### 4.1 Loans

([#6](https://github.com/uf-cpe-design-lab/stock/issues/6), amended by [#16](https://github.com/uf-cpe-design-lab/stock/issues/16))

1. A Lab User requests a Loan from an item or Location page, picking a quantity and optionally a Project they're on. Equipment is requested by Catalog Item, not by Unit.
2. Lab Staff approve or deny it. Approval reserves the count.
3. Staff fulfill it into the Team Cubby (or leave it out). Equipment: staff pick the Unit.
4. Return: staff check each unit in as Returned, Write-off (damaged) or Write-off (lost).

The **Project page** lists the Team, the Cubby and all open Loans.

### 4.2 Restocking

([#7](https://github.com/uf-cpe-design-lab/stock/issues/7))

- **Phone path:** scan a label with the in-app scanner to open the Location (or item) page, then tap **Running low** or **Out**, with an optional note. No sign-in is needed. A name is optional, and repeat taps are rate-limited. If the Location holds several items, the page asks which one.
- **Scan Session** (staff page, USB handheld): pick a mode, scan labels, review the list, then submit.
  - **Needs restock:** each scanned item gets flagged. Duplicate scans are ignored.
  - **Count:** scan a Location, then type the on-hand count for each item there.
- **Restock list:** one Restock Need per item, showing its reports together, with Mark ordered / Mark restocked / Dismiss.
- **Not in scope:** automatic reorder thresholds (§9).

### 4.3 Purchase Requests

([#8](https://github.com/uf-cpe-design-lab/stock/issues/8))

**The form, in order:**
1. Certify "I checked the lab doesn't already have this".
2. Line Items: quantity, product, unit price.
   - The product comes from a vendor search (Digi-Key, Mouser, Adafruit), a pasted vendor link (auto-filled where the vendor has an API), or free text.
   - A non-preferred vendor needs a reason and shows a *Non-preferred vendor* badge.
3. Purpose (required); Project and needed-by date (optional).
4. Agree to the terms. Lab Staff can edit the text; it currently covers the process and timeline (about 5 business days to resolve, 3–5 days shipping) and the Design 1 & 2 responsibility clause.

**Review:**
- The system flags Line Items that match an existing Catalog Item, to both the requester and the approver.
- Lab Staff approve or deny each line, or send the whole request back as *Needs changes*.

**Ownership:** everything bought is lab-owned. Project lines arrive as stock already on Loan to the requester. A rare "requester keeps it" exception is a staff-only option.

**Spend:** line and request totals, plus spend reports by Project and Course per Semester. No budgets, and no Reusable / Pooled classification.

### 4.4 Ordering and receiving

([#8](https://github.com/uf-cpe-design-lab/stock/issues/8), [#19](https://github.com/uf-cpe-design-lab/stock/issues/19))

- **The Order List** groups, per vendor, all approved Line Items not yet ordered and all Open Restock Needs whose Catalog Item uses that vendor.
  - Export: a cart / BOM CSV in Digi-Key's and Mouser's upload formats, or a list of links for the other vendors.
  - **Mark ordered** moves every line and Need in the batch to Ordered.
  - Ordering itself happens outside the system, through UF channels.
- **Receiving** happens from the batch. Staff enter the received quantity per line (and the count per counted Restock Need).
  - New products must be linked to, or created as, a Catalog Item before receiving. Their labels are printed at this point.
  - Project lines create an automatic **Out Loan** to the requester, fulfilled into the Team Cubby, and the requester is emailed.
  - Any remainder stays Ordered until it arrives or staff cancel it.

### 4.5 Catalog upkeep

([#5](https://github.com/uf-cpe-design-lab/stock/issues/5), [#3](https://github.com/uf-cpe-design-lab/stock/issues/3), [#20](https://github.com/uf-cpe-design-lab/stock/issues/20), [#22](https://github.com/uf-cpe-design-lab/stock/issues/22))

- **Categories:**
  - The tree is organized by function and editing it needs a Permission.
  - Starting top level: Passives, Semiconductors, Compute, Sensors & Modules, Displays, Motors & Actuators, Power & Batteries, Prototyping, Wire & Cables, Connectors, Hardware & Materials, Tools, Bench Equipment.
  - The full starting tree is on branch `prototype/taxonomy`.
- **Vendor lookup:**
  - Creating a Catalog Item from a vendor link fills in name, part number, link and image **once**. There's no bulk import or scheduled refresh, to stay within the Digi-Key and Mouser API terms.
  - The Lab Manager registers the Digi-Key and Mouser API accounts.
- **First inventory pass:** done cabinet by cabinet in the app: create Locations, add Catalog Items and Placements with counts, add Units, print labels. No CSV importer.

---

## 5. Locations, labels and scanning

([#23](https://github.com/uf-cpe-design-lab/stock/issues/23), [#9](https://github.com/uf-cpe-design-lab/stock/issues/9), [#4](https://github.com/uf-cpe-design-lab/stock/issues/4))

- **IDs** are short and readable aloud: cabinet `A`, drawer `A-07`, shelf bin `B-1-03`, bench `BENCH-4`. Units keep their lab numbers.
- **Labels identify the container** (Location) or the Unit, never the Catalog Item. A Location's page lists what it holds.
- **QR payload:** a type prefix plus the ID, with **no URL** because the domain may change: `L:A-07` for a Location, `U:SCOPE-01` for a Unit. The printed text shows `A-07`, the contents or name, and the site's short name.
- **In-app scanner:** a Scan button on every page opens the browser camera and routes `L:` and `U:` payloads to the matching page. Scan Sessions read the same payloads from the USB handheld (keyboard-wedge input into a focused field).
- **Stock:** one roll for everything, **62 mm continuous paper (DK-2205)**, cut to length per container type (short for drawers and bins, longer for cabinets and benches, a Unit-tag layout). The QR is about 10 mm. That's an estimate; check it when labels are first printed.
- **Printing:** the app queues Print Jobs, and a **lab print agent** (§7) prints them. Print failures aren't tracked; staff press **Reprint**.

---

## 6. Notifications and reports

([#18](https://github.com/uf-cpe-design-lab/stock/issues/18), [#21](https://github.com/uf-cpe-design-lab/stock/issues/21))

- **Channels:** email plus an in-app activity list.
- **Lab Staff** get an email **per event**: new Loan request, new Purchase Request, new Restock Report.
- **Lab Users** get an email when:
  - a Loan is approved, denied, fulfilled (with the Cubby) or cancelled;
  - a Purchase Request changes status;
  - a purchased item arrives.
- **Due-date reminders:** 3 days before, on the due date, then weekly while overdue.
- **Dashboard** (Lab Staff): work queues (Loans to approve, Loans to fulfill, Purchase Requests to review, open Restock Needs) and overdue Loans. Spend has its own report page.
- **Every table exports to CSV.**

---

## 7. Architecture and operations

([#15](https://github.com/uf-cpe-design-lab/stock/issues/15), [#24](https://github.com/uf-cpe-design-lab/stock/issues/24), [#25](https://github.com/uf-cpe-design-lab/stock/issues/25))

```text
off-campus Docker host                          lab LAN
┌───────────────────────────────────────┐       ┌───────────────────────────┐
│ caddy (TLS) ─▶ web (Django, gunicorn) │◀─HTTPS─│ print agent (e.g. a Pi)   │
│                worker (task queue,    │  poll  │   brother_ql_next ─▶ :9100│
│                  scheduler)           │       │   QL-820NWB               │
│                postgres               │       └───────────────────────────┘
│                mailpit (for now)      │
└───────────────────────────────────────┘
```

- **App:** Django (current LTS), PostgreSQL, server-rendered templates with htmx, dependencies managed with uv. Django's admin serves as the Lab Staff back office wherever it fits.
- **Worker:** a Postgres-backed task queue (no Redis) for emails and Print Jobs, plus a daily scheduler for reminders and overdue flags.
- **Email:** **Mailpit** for now. Production goes through the UF relay (`smtp.ufl.edu:587` with an ECE IT service account) once that's confirmed ([#25](https://github.com/uf-cpe-design-lab/stock/issues/25), deferred). "Relay or nothing": there's no third-party email fallback.
- **Print agent:** a small Python program on a lab machine.
  - It checks the app's API for queued Print Jobs over HTTPS, authenticating with a token.
  - It renders the jobs with `brother_ql_next` and sends them to the printer's port 9100.
  - Nothing connects into the lab from outside.
- **Hosting:** off campus on any Docker host with a public IP (provider undecided), with Caddy handling TLS.
  - The intended hostname is under `ece.ufl.edu`, pending ECE IT.
  - Nothing in the app may hard-code the domain; labels don't contain it.
- **Secrets:** a `.env` on the server (Django secret key, vendor API keys, print-agent token, SMTP credentials).
- **Backups:** a nightly `pg_dump` kept locally on the server.
- **Deploy:** GitHub Actions builds an image to `ghcr.io`, and the server runs `docker compose pull && docker compose up -d`.

---

## 8. Accepted risks

- **UF policy** ([#17](https://github.com/uf-cpe-design-lab/stock/issues/17), [#24](https://github.com/uf-cpe-design-lab/stock/issues/24)):
  - Loan records tie student names to courses, which is probably **Restricted (FERPA)** data.
  - UF policy 12-017 requires university-contracted hosting, and 12-025 requires GatorLink SSO for externally hosted apps.
  - Hosting off campus with magic-link sign-in **knowingly deviates** from both.
- **Local-only backups:** losing the server loses the data.
- **Network printing has no status read-back:** a silent failure means staff reprint.

## 9. Out of scope / deferred

- Automatic reordering at a minimum count ([#7](https://github.com/uf-cpe-design-lab/stock/issues/7)).
- GatorLink SSO ([#14](https://github.com/uf-cpe-design-lab/stock/issues/14)).
- The QR print-and-scan size test ([#12](https://github.com/uf-cpe-design-lab/stock/issues/12)).
- Confirming the UF relay and hostname with ECE IT ([#25](https://github.com/uf-cpe-design-lab/stock/issues/25)).
- Budgets and funding sources.
- A bulk CSV importer.
- Syncing Projects from the Google Sheet.
- Deployment, data entry and running the system: this spec ends at "ready to build".
