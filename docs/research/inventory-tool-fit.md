# Which open-source inventory tools fit the lab?

Research for [#2](https://github.com/uf-cpe-design-lab/stock/issues/2) (map [#1](https://github.com/uf-cpe-design-lab/stock/issues/1)). Checked 2026-10-26 against primary sources: official docs, source code on the default branch, release notes, and maintainer comments on issues and PRs. Domain terms follow `GLOSSARY.md`.

**Tags:** **[V]** means verified at the linked source. **[I]** means inferred from the source or the absence of a feature, and not tested on a running instance. None of the tools was installed for this research.

## Answer in one paragraph

No tool covers the lab end to end. Two workflows are missing everywhere: a Lab User raising a **Restock Report**, and a Lab User submitting a **Purchase Request** that an Admin approves. Every option would need custom work or a side tool for those. Three tools lead:

- **Shelf.nu** has the best workflow fit. It models all three kinds of stock natively and has Admin-approved booking requests by count. Its weak point is hosting: it depends on Supabase, and its Docker image is community-maintained.
- **InvenTree** is the strongest platform. It has the best auth, Brother QL-820NWB printing and a rich plugin system. But its maintainers declared Loans out of scope, so Loans, Restock Reports and Purchase Requests would all be custom plugin work.
- **Snipe-IT** sits in between. Its v8.8 request queue gets close to a Loan flow, but it has no Purchase Requests, and labels and scanning work only for Assets.

Homebox and Part-DB fall short on Loans and roles, and Part-DB also on Equipment.

## Scorecard

● = native fit, ◐ = partial or workaround, ○ = missing (custom build needed).

| Criterion | InvenTree 1.6.1 | Snipe-IT 8.8.3 | Shelf.nu 2.4.0 | Part-DB 2.20.3 | Homebox 0.27.1 |
|---|---|---|---|---|---|
| Three stock kinds | ● | ◐ | ● | ◐ (no Equipment units) | ◐ |
| Loans (request → Admin approve, due date, course/project) | ○ | ◐ | ● (course/project as free text only) | ○ | ○ |
| Restock Reports (phone QR, then batch USB scan) | ◐ | ◐ (Assets only) | ◐ | ◐ | ○ |
| Purchase Requests | ◐ (POs, no request/approval) | ○ | ○ | ◐ (vendor search, no workflow) | ○ |
| Hierarchical locations | ● | ● | ● | ● | ● |
| QR labels / Brother QL-820NWB | ● (official plugin) | ◐ (Assets only, PDF) | ◐ (QR zip + CSV) | ◐ (PDF) | ◐ (shell print hook) |
| Docker self-hosting | ● | ● | ◐ (needs Supabase) | ● | ● |
| Auth for GatorLink (SAML/Shibboleth) | ● | ◐ (accounts must already exist) | ◐ (unproven self-hosted) | ◐ (beta SAML) | ◐ (OIDC only) |
| Admin vs Lab User roles | ◐ (coarse) | ● | ● | ● | ○ |
| Extensibility | ● (plugins + API) | ◐ (API + webhooks) | ○ (no public API found) | ◐ (API) | ◐ (API) |
| Health / license | ● MIT, ~180 contributors | ● AGPL, very active | ● AGPL (mobile app not OSS) | ◐ AGPL, one maintainer | ◐ AGPL, pre-1.0 |

## Gaps every tool shares

- **No Lab-User Restock Report.** In every tool, flagging "low/out" requires stock edit rights or an Admin-side minimum-stock alert. A "scan → report low" page needs building, as a plugin or as a small front end over the API. [I]
- **No batch-scan processing.** Every tool accepts a USB keyboard-wedge scanner as typed input, but none has a "scan many, then process as restock updates" queue. InvenTree's multi-scan page has its batch action wired to `notYetImplemented`. [V, see InvenTree §3]
- **No Purchase Request approval with a "to requester vs. into stock" outcome.** At best a tool has Purchase Orders that receive into stock. [V/I per tool]
- **No Loan custom fields.** Course and project go into a free-text note, description or tags. InvenTree is the exception: its order Parameters could hold them. [V/I per tool]

---

## Shelf.nu

Repo [Shelf-nu/shelf.nu](https://github.com/Shelf-nu/shelf.nu). Release shelf@2.4.0 on 2026-10-23, about 3k stars, last push 2026-10-26. [V: `gh release list`, `gh api`]

- **Stock kinds.** `AssetType` is `INDIVIDUAL` or `QUANTITY_TRACKED`. Quantity-tracked assets have a `ConsumptionType` of `ONE_WAY` (consumed) or `TWO_WAY` (returned), plus `minQuantity` and `unitOfMeasure`. A ledger records checkout, return, restock, consume, loss and damage. These map directly onto Consumable (ONE_WAY), Loanable (TWO_WAY) and Equipment (INDIVIDUAL). [V: [schema.prisma](https://github.com/Shelf-nu/shelf.nu/blob/main/packages/database/prisma/schema.prisma)] Whether a Consumable can skip counting entirely is [I].
- **Loans.** A booking moves DRAFT → RESERVED → ONGOING → OVERDUE → COMPLETE, with a start and end, so it has a due date and an overdue state. Each booked asset carries a quantity, and a "model request" books N units of a model without picking specific units. Base users "can browse gear and submit booking requests; an admin approves and hands it over". They cannot check out themselves, and only an Administrator or Manager completes the hand-off. [V: schema; [roles KB](https://www.shelf.nu/knowledge-base/user-roles-and-their-permissions)] Bookings have name, description and tags but no custom fields, so course and project go into free text or tags. [I from schema]
- **Restock.** `minQuantity` gives a low-stock threshold. There is no Lab-User report flow. [V schema / I absence]
- **Scanning.** The scanner has a camera mode and a keyboard-input "scanner" mode, so a USB handheld should work. [V: `apps/webapp/app/components/scanner/code-scanner.tsx`] QR codes encode `<SERVER_URL>/qr/<id>`, which opens the item page. [V: `apps/docs/labels-export.md`]
- **Purchase Requests.** None. [I]
- **Locations and labels.** Locations nest (`parentId`). A bulk QR zip includes `labels.csv` for third-party label software, which is the path to a Brother QL (for example P-touch Editor). Barcode types beyond QR are a paid feature on the hosted service, but self-hosting unlocks everything by setting `ENABLE_PREMIUM_FEATURES=false`. [V: schema, labels-export.md, `.env.example`; QL path I]
- **Self-hosting is the main risk.** The docs say: "The Docker configuration for shelf.nu is an effort powered by people within the community … Use at your own risk", and "our docker setup doesn't currently support self-hosting supabase". The example points at Supabase Cloud. [V: [apps/docs/docker.md](https://github.com/Shelf-nu/shelf.nu/blob/main/apps/docs/docker.md)] Running self-hosted Supabase (its own multi-container stack) underneath is possible in principle, but neither project documents it for Shelf. [I]
- **Auth.** SAML runs through Supabase Auth, with group-to-role mapping and a dedicated Shibboleth provider guide. The SSO docs say SAML is "for Team and Enterprise plan customers. Please contact Sales". Self-hosters must configure SAML in their own Supabase. [V: [sso/index.md](https://github.com/Shelf-nu/shelf.nu/blob/main/apps/docs/sso/index.md), `sso/providers/shibboleth.md`] Whether that works on self-hosted Supabase is [I, needs a test].
- **Roles.** Owner, Administrator, Manager, Self-service and Base, with a permission matrix and visibility toggles. [V: `apps/docs/roles-and-permissions.md`, roles KB]
- **Extensibility.** The repo has internal web and mobile API routes but no documented public API. No plugins. [I]
- **License.** AGPL-3.0, except the mobile app `apps/companion/`, which is under PolyForm Internal Use (source-available, not OSS). [V: LICENSE files]

## InvenTree

Repo [inventree/InvenTree](https://github.com/inventree/InvenTree). 1.6.1 released 2026-10-21 (1.0.0 was 2025-09-14). MIT, about 7.8k stars, about 180 contributors, about 362 commits in the last 3 months. [V: `gh api`, releases]

- **Stock kinds.** A Part is a kind of item. A StockItem is a quantity of it at a Location, so every StockItem has a quantity; Consumables can simply go uncounted. [V: [docs/part](https://docs.inventree.org/en/stable/part/)] A `consumable` flag (1.5.0) affects BOMs and builds only. [V: [part/consumable](https://docs.inventree.org/en/stable/part/consumable/)] Equipment would be a **Trackable** part with one serialized StockItem per unit, with the lab number as the serial. [V: [part/trackable](https://docs.inventree.org/en/stable/part/trackable/)] A unit at a bench is a StockItem in a bench Location. [I]
- **Loans: out of scope.** Maintainers closed "[FR] Rental support" as not planned, saying it is "outside the scope of the core InvenTree functionality" and suggesting a plugin. [V: [#9068](https://github.com/inventree/InvenTree/issues/9068)] The workarounds don't fit well:
  - Sales Order out plus Return Order back requires a customer Company per borrower. [V: `stock/serializers.py` `StockAssignmentSerializer`]
  - A Transfer Order (1.4.0) to an "On loan" location has a target date with an overdue flag, a Responsible, a project code and notes. [V: [stock/transfer_order](https://docs.inventree.org/en/stable/stock/transfer_order/); fit as a Loan I]
  - Course and project could be Parameters on orders. [V: `order/models.py` uses `InvenTreeParameterMixin`]
  - There is no request permission or approval step. A `TransitionMixin` plugin can intercept state changes, which is the hook for building approval. [V: [plugins/mixins/transition](https://docs.inventree.org/en/stable/plugins/mixins/transition/)]
- **Restock.** Parts have a minimum stock level, with "low stock" notifications to subscribed users in the app and by email. [V: [part/notification](https://docs.inventree.org/en/stable/part/notification/)] The internal barcode format is JSON or `INV-SI123`, not a URL, but label templates get a `qr_url` variable, so labels can carry URL QR codes that a phone opens. [V: [barcodes/internal](https://docs.inventree.org/en/stable/barcodes/internal/); `qr_url` in `part/models.py`, `stock/models.py`] The web UI accepts webcam and keyboard-wedge input. The multi-scan page's only batch action ("Count") calls `notYetImplemented`. [V: `src/frontend/src/pages/Index/Scan.tsx`] The mobile app takes wedge scanners (they must send Enter after each code). [V: [app/barcode](https://docs.inventree.org/en/stable/app/barcode/)] A Lab User can't flag low stock without Stock change permission. [I]
- **Purchase Requests.** Purchase Orders run Pending → Placed → Complete, with an optional Responsible and a default receive location, and Supplier Parts carry a product URL. [V: `order/status_codes.py`, `company/models.py`] "[FR] Purchase Order Approvals" is still open; a maintainer points to a closed-source approval plugin from his company, invenhost. [V: [#3501](https://github.com/inventree/InvenTree/issues/3501)] Received items always become stock. [I]
- **Locations.** Hierarchical, with structural locations, location types and location barcodes. [V: [docs/stock](https://docs.inventree.org/en/stable/stock/)]
- **Labels.** HTML templates are rendered to PDF, and label printers are configured as "machines". The official [inventree-brother-plugin](https://github.com/inventree/inventree-brother-plugin) (v2.0.6, May 2026) lists the **QL-820NWB** and prints over the network or USB. [V]
- **Docker.** The official compose file runs Postgres, Redis, server, worker and a Caddy proxy; plugins install through `plugins.txt`. [V: `contrib/container/docker-compose.yml`, [start/docker](https://docs.inventree.org/en/stable/start/docker/)]
- **Auth.** Local accounts, LDAP (`django-auth-ldap`), REMOTE_USER header login, MFA (TOTP, WebAuthn) and, since 1.6.0, SCIM provisioning. [V: `InvenTree/settings.py`; 1.6.0 notes] SSO runs through django-allauth: OIDC via its `openid_connect` provider (the docs include a Keycloak OIDC example and IdP group sync) and **SAML login added in 1.6.0** ([#12799](https://github.com/inventree/InvenTree/pull/12799), closing "[FR] Add SAML login" [#3509](https://github.com/inventree/InvenTree/issues/3509)). [V: [settings/SSO](https://docs.inventree.org/en/stable/settings/SSO/), 1.6.0 notes] *Correction to a common reading:* the 1.6.0 note "enable OIDC/oAuth2 provider by default" ([#12731](https://github.com/inventree/InvenTree/pull/12731)) is InvenTree acting as an OAuth2 *server* for apps, not a login method. [V] No Shibboleth notes exist; GatorLink should work as a generic SAML IdP or through Apache `mod_shib` with REMOTE_USER. [I]
- **Roles.** Groups get view/add/change/delete on 12 roles (part, stock, purchase order, and so on). There is no per-action right such as "may report low" or "may request". Item ownership only hides edit buttons. [V: [settings/permissions](https://docs.inventree.org/en/stable/settings/permissions/), `users/ruleset.py`, [stock/owner](https://docs.inventree.org/en/stable/stock/owner/)]
- **Extensibility.** The richest of all candidates. Plugin mixins include UI, URL, API, navigation, event, schedule, barcode, label, machine, transition, validation and notification, and 1.6 lets plugins add custom pages. There is a full REST API with a Python client, plus model metadata. [V: `docs/docs/plugins/mixins/`] No loan, request or kiosk plugin exists in the official plugin list. [V: inventree-website `_repo`]

## Snipe-IT

Repo [grokability/snipe-it](https://github.com/grokability/snipe-it). v8.8.3 released 2026-10-22. AGPL-3.0, about 15k stars, about 650 commits in the last 30 days. Hosted plans fund it. [V: `gh api`, releases, [pricing](https://snipeitapp.com/pricing)]

- **Stock kinds.** Assets (individually tagged) fit Equipment. Accessories ("checked out … and checked back in", by quantity) fit Loanables. Consumables ("do not get checked back in") fit Consumables. [V: [overview](https://snipe-it.readme.io/docs/overview)] Accessory checkout takes `checkout_qty` and has bulk check-in. [V: `AccessoryCheckoutController.php`] Consumable `qty` is required (`min:0`), so counting is technically mandatory but can be left at 0. [V: `app/Models/Consumable.php`]
- **Loans.** The v8.8.0 request overhaul ([PR #19539](https://github.com/grokability/snipe-it/pull/19539), merged 2026-08-22) made all six item types requestable. A request has a quantity, optional reservation dates and a free-text note. Admins get a queue with bulk and partial fulfillment, and requests are Pending, Fulfilled or Canceled. [V] It is a queue, not an approval: approving means doing the checkout, and denying means cancelling. [I from PR] Expected check-in dates and overdue alerts exist for Assets. [V: [email-alerts](https://snipe-it.readme.io/docs/email-alerts)] Accessory checkouts store only a note, with no due date. [I: `AccessoryCheckout.php`] Custom fields are on Asset models only, but can appear on the checkout form and are logged with the checkout. That covers course and project for Equipment; for Accessories only the note is available. [V: [custom-fields](https://snipe-it.readme.io/docs/custom-fields), `Traits/Loggable.php`]
- **Restock.** Minimum quantities and a daily low-inventory email cover Accessories, Consumables and Components, and v8.7 added a logged replenish action. [V: [notifications](https://snipe-it.readme.io/docs/notifications), PR #19392] **Labels, QR codes, the mobile app and quick-scan are all Asset-only.** The only label route is `label/hardware`, and a request for non-asset labels was closed without action. [V: `routes/web/hardware.php`, [#16582](https://github.com/grokability/snipe-it/issues/16582)] Quick-scan audit, quick-scan check-in and bulk checkout support USB scanners for Assets. [V]
- **Purchase Requests.** None. Items have supplier, order number and cost fields only. [V/I]
- **Locations.** Nested via `parent_id`. Assets and Accessories can be checked out to a location, such as a bench. [V: `app/Models/Location.php`]
- **Labels.** PDF generation from asset lists. Templates exist for Brother **TZe** tapes, Dymo, Zebra and sheet labels, but not Brother QL/DK. v8.8.1 added a custom label editor for arbitrary sizes, which probably covers QL. [V: `app/Models/Labels/`, PR #18938; QL I]
- **Docker.** Official `snipe/snipe-it` image and compose file. [V: [docker](https://snipe-it.readme.io/docs/docker)]
- **Auth.** Local, LDAP/AD (with sync), SAML, Google, REMOTE_USER, TOTP and SCIM. The SAML docs cover Shibboleth (eduPersonPrincipalName, `requestedAuthnContext=false`), but **SAML only logs in users who already exist**, so accounts must come from LDAP, SCIM or CSV import first. [V: [saml](https://snipe-it.readme.io/docs/saml), `app/Services/Saml.php`] There is no OIDC web login; the issue was closed against a PR that only accepts OIDC tokens on the API. [V: [#12695](https://github.com/grokability/snipe-it/issues/12695), [PR #19378](https://github.com/grokability/snipe-it/pull/19378)]
- **Roles.** Granular per-type permissions for view, create, edit, checkout and checkin, plus `self.*` and requestable rights. A self-service user sees their own items and the requestable catalog. [V: `config/permissions.php`]
- **Extensibility.** Broad REST API, plus checkout and check-in webhooks (Slack, Teams, Google Chat, generic). No plugin system. [V: [reference](https://snipe-it.readme.io/reference), [webhooks](https://snipe-it.readme.io/docs/webhook-integration)]

## Part-DB

Repo [Part-DB/Part-DB-server](https://github.com/Part-DB/Part-DB-server). v2.20.3 released 2026-10-25, eight releases in five weeks. AGPL-3.0, about 1.8k stars. **Bus factor is about 1**: one maintainer wrote about 556 of about 600 commits in the last six months. [V: `gh api` contributors and commits]

- **Stock kinds.** Parts hold Part Lots, each with location, amount, an "amount unknown" flag, `needs_refill`, owner and barcode. This suits Consumables (amount unknown) and Loanables (counted). [V: [concepts](https://docs.part-db.de/concepts.html), `PartLot.php`] There is no serial or asset concept; Equipment would be one Part or one Lot per unit, which is a workaround. [I]
- **Loans.** None: no checkout, due date or approval. Lot owners (only the owner may withdraw) plus moving stock to an "On loan" location is a weak stand-in. [V: [tips_tricks](https://docs.part-db.de/usage/tips_tricks.html), `PartLotVoter.php`; workaround I]
- **Restock.** The best built-in scanner page of the five. It accepts typed input, keyboard-wedge USB scanners and the camera, and a SOH-prefix lets a scanner jump to the item from any page. It also parses Digi-Key, Mouser, LCSC and TME distributor barcodes. QR codes encode `/scan/{part|lot|location}/{id}`, so a phone opens the item. [V: [scanner](https://docs.part-db.de/usage/scanner.html), `BarcodeContentGenerator.php`] Min-stock is documented as "not fully implemented". `needs_refill` is set in the edit form, so it likely needs edit rights. [V: concepts; I]
- **Purchase Requests.** No workflow ([#1427](https://github.com/Part-DB/Part-DB-server/issues/1427), [#217](https://github.com/Part-DB/Part-DB-server/issues/217) are open). It has the best vendor search: info providers for Digi-Key, Mouser, LCSC, TME, Octopart and others create parts from a vendor search. That is relevant to Preferred Vendors ([#3](https://github.com/uf-cpe-design-lab/stock/issues/3)). [V: [info providers](https://docs.part-db.de/usage/information_provider_system.html)]
- **Locations and labels.** Hierarchical storage locations. Labels are PDFs from profiles with placeholders or Twig; there is no direct Brother QL driver. [V: [labels](https://docs.part-db.de/usage/labels.html), [#489](https://github.com/Part-DB/Part-DB-server/issues/489)]
- **Docker, auth and roles.** Official image. Local login with TOTP and WebAuthn. SAML is **beta**, tested only with Keycloak, and uses NameID as the username. There is no LDAP (the maintainer says to use SAML through Keycloak) and no OIDC login yet. Permissions are very granular: withdraw, add and move are separate from edit. [V: [installation_docker](https://docs.part-db.de/installation/installation_docker.html), [saml_sso](https://docs.part-db.de/installation/saml_sso.html), [#590](https://github.com/Part-DB/Part-DB-server/issues/590), [#593](https://github.com/Part-DB/Part-DB-server/issues/593), `config/permissions.yaml`]
- **API.** API Platform REST API with scoped tokens and an event log. No plugins. [V: [api/authentication](https://docs.part-db.de/api/authentication.html)]

## Homebox

Use the **sysadminsmedia/homebox** fork. The original hay-kot/homebox is archived (last push 2024-06). The fork is at v0.27.1 (2026-10-23), AGPL-3.0, about 7.6k stars, pre-1.0, and v0.26 made a breaking API and database change. [V: `gh api`, releases]

- **Stock kinds.** Everything is an "entity" (v0.26), with a decimal quantity, asset ID, serial number, custom fields and user-defined Entity Types with templates. [V: `schema/entity.go`, [entity-types](https://homebox.software/en/user-guide/entity-types/)]
- **Loans.** None. Lending requests [#395](https://github.com/sysadminsmedia/homebox/issues/395) and [#74](https://github.com/sysadminsmedia/homebox/issues/74) are unimplemented. [V]
- **Restock.** No low-stock alerts; [PR #1686](https://github.com/sysadminsmedia/homebox/pull/1686) is open. Label QR codes open `/item/{id}`, and there is a camera scanner. [V: `v1_ctrl_labelmaker.go`]
- **Purchase Requests.** None. [I]
- **Labels.** `HBOX_LABEL_MAKER_PRINT_COMMAND` runs a server-side command on the label PNG, so `brother_ql` could print to the QL-820NWB. [V: [configure](https://homebox.software/en/quick-start/configure/); QL I]
- **Auth.** Local accounts and OIDC (with a group allow-list). No SAML or LDAP. [V: [oidc](https://homebox.software/en/quick-start/configure/oidc/)]
- **Roles.** Disqualifying: **every group member can create, edit and delete everything**. The only roles are `user` and `owner`, and owner only covers group administration. [V: `schema/user_group.go`, `routes.go`, [#1714](https://github.com/sysadminsmedia/homebox/issues/1714)]

## Others checked

| Tool | Verdict | Why |
|---|---|---|
| **LibreBooking** (GPL-3.0, v7.1.0, 2026-10-13) | Complement only | Per-resource "require approval", bookable accessories with quantity limits, custom attributes on reservations, and LDAP/CAS/SAML/OAuth2 auth. But it is a reservation calendar with no stock or QR. Relevant only if the chosen tool lacks Loan approval. [V: docs/source/ADMINISTRATION.rst] |
| **GLPI** (GPL-3.0, 12.0.0) | Exclude | Heavyweight IT service desk. Reservations are per item with no approval, consumables are allocated by admins, and OAuth SSO is a paid plugin. [V: glpi-user-documentation, help.glpi-project.org] |
| **eLabFTW** (AGPL-3.0, 5.6.15) | Exclude | Lab notebook with storage inventory and a scheduler, but no loan approval. Strong native SAML. [V: doc.elabftw.net] |
| **Ralph**, **Grocy**, **OpenBoxes**, **Fab Manager** | Exclude | No count-based Loans with approval: data-center assets, groceries, supply chain and fablab billing respectively. [V/I] |
| Lend Engine | Gone | The repo returns 404. Searches for "equipment checkout" and "tool library" found only toy projects. [V] |

## What this means for "Adopt which tool, or build?" ([#10](https://github.com/uf-cpe-design-lab/stock/issues/10))

Whatever is adopted, the lab will build something. The real choice is how much and on which base:

1. **Shelf.nu nearly as-is.** Loans and stock kinds work out of the box. Restock Reports and Purchase Requests would live in a side tool, since there is no public API to extend. Viable only if the lab accepts running Supabase (self-hosted, which is undocumented, or cloud, which breaks "self-hosted") and a community Docker image.
2. **InvenTree plus a lab plugin.** Use the UI/URL/Transition mixins to add a Lab-User page for Loan requests, Restock Reports and Purchase Requests. This needs the most custom code, but it sits on the most robust and extensible base, with QL-820NWB printing and SAML/LDAP already solved.
3. **Snipe-IT plus a small companion app** on its REST API for Restock Reports and Purchase Requests. Loans are mostly covered by the request queue. The main friction is Asset-only labels and the pre-provisioning that SAML needs.

## Open questions this raises

- **How does GatorLink federate?** Does UF offer only SAML (Shibboleth), or OIDC too? Will UF register a lab service and release `eduPersonPrincipalName` and group attributes? Is UF LDAP reachable for account provisioning? This decides auth for every candidate. Snipe-IT needs pre-provisioned accounts, Homebox needs OIDC, Part-DB's SAML is beta, and Shelf's is unproven self-hosted.
- **Is Supabase acceptable as a dependency?** If not, Shelf.nu is out.
- **Do Restock Reports need a login?** If anonymous scan-to-report is acceptable, a tiny public endpoint avoids giving Lab Users stock edit rights in any tool.
- **Should Purchase Requests live in the inventory tool at all?** No candidate has them. A form or ticket flow that hands off to UF procurement may be simpler.
