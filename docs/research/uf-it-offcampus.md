# What will UF IT provide for an off-campus host?

Research for [#17](https://github.com/uf-cpe-design-lab/stock/issues/17) (map [#1](https://github.com/uf-cpe-design-lab/stock/issues/1)). Checked 2026-10-26 against UFIT pages (it.ufl.edu, mail.ufl.edu, the UFIT TeamDynamix portal), UF policies (policy.ufl.edu), Herbert Wertheim College of Engineering (HWCOE) IT pages (it.eng.ufl.edu), and live DNS and SMTP probes.

**Tags:** **[V]** means verified at the linked source or by a live probe. **[I]** means inferred, or concluded because nothing says otherwise. Nothing was requested from UF IT, and no credentials were tried.

**What could not be seen:** the Infoblox IPAM UI (`ipam.name.ufl.edu`), the CERTInext portal, the UF Integrated Risk Management (IRM) intake, the HWCOE IT SharePoint list of unit ISMs, and the TeamDynamix ticket forms all need a GatorLink login. The UF Help Desk wiki page that used to document `smtp.ufl.edu` client settings (`kb-test.helpdesk.ufl.edu`) no longer resolves.

**Probe caveat:** the machine that ran the probes was **on the UF network** (public IP 128.227.3.226). So "port 587 is open" here does **not** prove that `smtp.ufl.edu` accepts connections from off campus.

## Answer in one paragraph

The technical pieces exist, but **UF policy is the real constraint, and it changes the plan.** **Email:** UF still runs its own authenticated mail submission service, `smtp.ufl.edu` (port 587, STARTTLS, `AUTH PLAIN LOGIN`, listed in UF's public DNS). Department IT can request a **service account with a mailbox and a hyphenated `@ufl.edu` alias** (e.g. `ece-stock@ufl.edu`). An app logging into `smtp.ufl.edu` as that account is the most plausible way to get a "UF relay". **No UFIT page documents application or off-campus use, though, so ECE IT and UFIT have to confirm it.** Microsoft 365 SMTP AUTH is controlled by UF's tenant admins. Basic auth goes off by default at the end of December 2026, and OAuth would need UFIT to register an Entra app, which no UF page offers. `ufl.edu` and `ece.ufl.edu` both publish **DMARC `p=quarantine`**, so mail "from" a UF address that bypasses UF's servers goes to Junk. **Hostname:** `ece.ufl.edu` sits on UFIT's central DNS (Infoblox). UF names can and do point off campus (`giving.ufl.edu` → GiveCampus/Cloudflare). ECE IT requests the record, but UFIT's web best practices say off-site services need "appropriate contracts". **TLS:** there are no CAA records, so Let's Encrypt HTTP-01 works. UF's own free option is **CERTInext (InCommon), which supports ACME** (certbot, acme.sh) through unit IT, with lifetimes of 200 days now and shrinking. **Data policy:** student names tied to courses and loans are very likely **non-directory education records → Restricted (FERPA)**. UF policy then requires a **UF risk assessment** (IRM), **documented approval from Senior Leadership and the CIO**, and use of **"university enterprise IT services whenever available"**. External hosting must be "approved, contracted, and managed by the university". Public cloud (AWS/Azure/GCP) must go through **UFIT's managed service**, and an IaaS vendor holding Restricted data needs a **Data Security Agreement**. HWCOE's own procedure bars third-party hosting unless UFIT offerings can't do the job *and* a risk assessment is done. And **externally hosted applications "must authenticate users using Gatorlink SSO"**, which works against magic-link sign-in. **Bottom line [I]: a lab-rented VPS with a public IP will not pass review. A UFIT-managed cloud account or a UFIT/HWCOE on-prem VM, either one with a risk assessment, is the realistic "off-campus-like" route. And sign-in should plan on GatorLink SSO, not magic links.**

## 1. Email relay from an off-campus host

### What exists

| Fact | Tag | Source |
|---|---|---|
| `smtp.ufl.edu` resolves to 128.227.36.85 (PTR `virtual-msa-prod01.osg.ufl.edu`), in the **external** DNS view too (answered by 8.8.8.8). | [V] | `dig smtp.ufl.edu`, `dig @8.8.8.8 smtp.ufl.edu` |
| On 587 it offers STARTTLS with an InCommon/Sectigo OV certificate (`CN=smtp.ufl.edu`, issuer "InCommon RSA OV SSL CA 3"). After EHLO it advertises `AUTH PLAIN LOGIN`, `SIZE 35000000` (~35 MB), `8BITMIME`, `PIPELINING`. Port 25 is open; 465 is closed. | [V] | `openssl s_client -starttls smtp -connect smtp.ufl.edu:587` from the UF network |
| UFIT's GatorMail service "also encompasses SMTP, and Proofpoint on-premise mail volume hygiene service." | [V] | [UFIT service: GatorMail](https://ufl.teamdynamix.com/TDClient/33/IT-Portal/Requests/Service/177/GatorMail) |
| No UFIT KB article covers SMTP, relays, or apps sending mail. A KB search for "smtp" returns 0 articles. | [V] | [TDX KB search](https://ufl.teamdynamix.com/TDClient/33/IT-Portal/KB/Search?SearchText=smtp), [GatorMail KB category](https://ufl.teamdynamix.com/TDClient/33/IT-Portal/KB/Category/166/GatorMail) |
| Older UF pages describe off-campus sending through `smtp.ufl.edu` on 587 with GatorLink credentials (Physics, 2006; a now-dead Help Desk wiki page). Current UFIT client guidance points people at Microsoft 365 instead. | [V that the pages say so; I that this still holds] | [UF Physics computing: email](https://phys.ufl.edu/computing/email.shtml), [Connecting to Email Applications](https://www.mail.ufl.edu/managing-email/connecting-to-email-applications/) |
| **Service accounts with mailboxes exist.** Dept IT can request an `@ufl.edu` alias for one if the service account already exists with a mailbox, the alias has a hyphen (e.g. `test-account@ufl.edu`), and the request comes from Dept IT. | [V] | [KB 538: Email Alias for Service Accounts](https://ufl.teamdynamix.com/TDClient/33/IT-Portal/KB/Article/538/Email-Alias-for-Service-Accounts) |
| "Units may request email accounts for administrative purposes … that are not tied to a specific GatorLink account." | [V] | [UFIT TSS Email Requirements](https://it.ufl.edu/tss/procedures/email-requirements/) |
| A UF **shared mailbox** has no password of its own: access "is granted after signing in as an authorized user to a UF mailbox." So a shared mailbox cannot log into SMTP by itself. | [V for the quote; I for the SMTP consequence] | [HWCOE IT: What is a shared mailbox?](https://it.eng.ufl.edu/?p=11823) |
| Inbound mail for `ufl.edu` and `ece.ufl.edu` goes to Exchange Online (`ufl-edu.mail.protection.outlook.com`, `ece-ufl-edu.mail.protection.outlook.com`). | [V] | `dig MX` |
| `ufl.edu` and `ece.ufl.edu` both publish `v=DMARC1; p=quarantine` and a Valimail dynamic SPF (`include:…_spf.vali.email ~all`). UFIT: since **March 10, 2026**, mail that fails authentication "as" any `@ufl.edu` address or subdomain goes to Junk. Third-party senders need UFIT to publish DKIM records. | [V] | `dig TXT _dmarc.ufl.edu`; [Third-party Bulk Email Services](https://www.mail.ufl.edu/getting-help/third-party-bulk-email-services/) |
| HWCOE IT's service links include GatorMail and OWA. No SMTP relay is listed. | [V] | [it.eng.ufl.edu](https://www.it.eng.ufl.edu/) |

### Microsoft 365 route

- `smtp.office365.com:587` SMTP AUTH is a tenant-level setting. UF's tenant admins (UFIT) control it. No UF page offers it, or Entra app registrations, to departmental apps. [I, from absence]
- Microsoft Message Center MC786329 (revised 2026-01-27): Basic-auth SMTP AUTH keeps working until the **end of December 2026**. It is then **disabled by default** for existing tenants, though admins can turn it back on. The final removal date is due in H2 2027. The documented way forward for a daemon is OAuth client credentials with `SMTP.SendAsApp` (an Entra app registration plus an Exchange service principal scoped to one mailbox), or Microsoft Graph `sendMail`. [V via a [mirror of MC786329](https://mc.merill.net/message/MC786329/compare/2024-04-26T00-17-12-6Z/2026-01-27T20-11-01-253Z), not Microsoft's own page]
- **[I]** Either way, UFIT has to create an app registration and grant it rights in UF's tenant. Expect that request to go through a risk assessment. This route is heavier than `smtp.ufl.edu`.

### What's required (the likely path) [I unless tagged]

1. ECE IT (`support@ece.ufl.edu`) requests a **UFAD service account with a mailbox** and the alias `ece-<name>@ufl.edu` [V that this kind of request exists, KB 538].
2. The app sends via `smtp.ufl.edu:587` with STARTTLS and logs in as that account, using `From: ece-<name>@ufl.edu`. Mail sent through UF's own server should pass SPF/DKIM/DMARC alignment. That isn't verified, since we can't see Valimail's per-IP rules.
3. **Confirm with UFIT/ECE IT** (none of this is documented):
   - Does `smtp.ufl.edu` accept connections from **non-UF IPs** today? It is in the external DNS view, which suggests yes, but that is untested.
   - Can a **service account log in to SMTP with only a password**, or is Duo/MFA (or Conditional Access) in the way?
   - Must the host's **IP be registered** or allow-listed? The old IFAS practice was to register SMTP servers ([IFAS 2004](https://icc.ifas.ufl.edu/ITServiceDocs/Email/exchange.htm)).
   - What are the **rate or recipient limits**? Only the 35 MB message size is known [V].
   - Whether the backend behind `smtp.ufl.edu` is staying or being retired as GatorMail moves further into Exchange Online.
4. Cloud providers often block outbound port 25. Port 587 is the one needed, so that block doesn't matter here. [I]

**Feasibility verdict [I]: "Relay or nothing" is plausible but not confirmed.** UF does run an authenticated submission host, and departments can get service accounts. Nothing published promises that an off-campus app may use it. One ECE IT ticket asking these questions settles it.

## 2. A `ufl.edu` hostname for an off-campus IP

| Fact | Tag | Source |
|---|---|---|
| `ece.ufl.edu` is served by UF's central nameservers (`ns`, `rns`, `ens.name.ufl.edu`), the same as `ufl.edu`. There is no separate departmental DNS. | [V] | `dig NS ece.ufl.edu` |
| DNS is run by **UFIT Infrastructure Reliability Services (IRS)** on Infoblox: ufit-ict-infrastructure-reliability-service@mail.ufl.edu, (352) 392-2061. Units can host their own zones on Infoblox by request. "Any non UF IP address will receive answers from the External view." | [V] | [UFIT DNS & DHCP Services](https://it.ufl.edu/infrastructure/dns--dhcp-services) |
| HWCOE IT links the Infoblox IPAM UI, which suggests college/department IT staff have delegated record-editing access. | [V for the link; I for the access] | [it.eng.ufl.edu](https://www.it.eng.ufl.edu/) |
| UF names already point off campus, e.g. `giving.ufl.edu` → CNAME `branded.givecampus.com` (Cloudflare IPs). So the external view accepts non-UF targets. | [V] | `dig @8.8.8.8 giving.ufl.edu` |
| Domain Name Policy 12-012: third-level names (`x.ufl.edu`) are requested by a dean or VP or their designee. "All official university web sites shall use domain names within the ufl.edu domain space." Nothing is said about hosting location. Amended 2024-07-18. | [V] | [Policy 12-012](https://policy.ufl.edu/policy/domain-name-policy/) |
| UFIT Web Services best practices: "Services should be hosted on UF servers." "Services may be hosted on external servers under appropriate contracts." "All domains must be recorded with CNS … regardless of hosting location." Services in the ufl.edu namespace "may be managed by non-UF staff under appropriate contracts." | [V] | [UFIT Web Services best practices](https://webservices.it.ufl.edu/about/best-practices) |

**Answer [I]:** Yes, technically. A name under `ece.ufl.edu` (a fourth-level name) is ECE IT's to request or create in Infoblox, and an A record to an off-campus IP is allowed. **ECE IT issues it.** Expect them to ask for the risk assessment and hosting approval from section 4 first, because the "appropriate contracts" language applies.

## 3. TLS certificate

| Option | Works? | Evidence |
|---|---|---|
| **Let's Encrypt HTTP-01** | Yes, technically. `ufl.edu` and `ece.ufl.edu` have **no CAA records**, so any public CA may issue. HTTP-01 only needs the A record to reach the host on port 80. | [V] `dig CAA` (empty, also via 8.8.8.8). No UF page found that forbids non-InCommon CAs [I, from absence]. |
| Let's Encrypt **DNS-01** | Unlikely. It needs write access to TXT records in UF's Infoblox, or a CNAME of `_acme-challenge.<name>` pointing at a zone we control (which ECE IT would have to create). | [I] |
| **UF CERTInext (InCommon)** | Yes, and it is free. Since **2026-07-17** UF certificates come from CERTInext. **ACME is supported** (certbot, win-acme, acme.sh), and `*.ufl.edu` names are pre-validated, so there is no per-request DCV. Wildcard OV is available. Certificates come through **college/departmental IT**, not UFIT IAM. Lifetimes: **200 days now, 100 days in 2027** (one passage says 90 days from Jan 2027), **47 days in 2029**. Contacts: IT-IAMSSLCertAdmin@ufl.edu, identity-services@it.ufl.edu. | [V] [UFIT SSL Certificates](https://it.ufl.edu/iam/authentication-systems/ssl-certificates); `smtp.ufl.edu`'s own certificate is InCommon OV [V] |

**Answer [I]:** Either one works from off campus. CERTInext via ACME (ECE IT supplies the account and EAB credentials) gives an InCommon OV certificate that matches UF's other services. Let's Encrypt HTTP-01 is the fallback that needs no extra request. Automate renewal whichever you choose.

## 4. UF policy on hosting university data off campus

### Classification

- Policy 12-011: all UF data is **Restricted**, **Sensitive** or **Open**. "Collections of diverse information should be classified as to the most secure classification level." The **data owner** (dean, director or department chair) approves the classification. Reviewed 2025-12-17. [V, [Policy 12-011](https://policy.ufl.edu/policy/data-classification-policy)]
- Classification guidelines table: **"Student records (non-directory) — Restricted — FERPA"**, while **"UF Directory (students & staff) — Open"**. Revised 2012-05-10. [V, [guidelines PDF](https://policy.ufl.edu/wp-content/uploads/2025/05/data-classification-guidelines.pdf)]
- UF directory information covers name, "class and college", email, enrollment status, majors and similar. **Course enrollment and course-level schedules are not on the list.** [V, [UF catalog: FERPA](https://catalog.ufl.edu/UGRD/academic-regulations/ferpa-confidentiality-student-records/)]
- **[I]** A loan record tying a student's name to a **course** is directly related to the student and maintained by UF. It is therefore a FERPA education record, and **non-directory**. Expect **Restricted**. Name and email alone (no course) would arguably be Open directory data. The ECE chair, as data owner, makes the final call.

### Hosting rules that apply

| Rule | Tag | Source |
|---|---|---|
| IT Rationalization Policy 12-017 (2023): units must "Use university enterprise IT services and infrastructure whenever available"; "Utilize external IT services (including 'cloud') that have been approved, contracted, and managed by the university"; and "Obtain documented approval from the UF Senior Leadership and the Vice President and Chief Information Officer before implementing information systems that store, process, or transmit sensitive or restricted data". That approval is documented in the IRM system. | [V] | [Policy 12-017](https://policy.ufl.edu/?p=7995) |
| Risk Management Policy 12-025: risk assessments happen **before acquiring** a system and after changes such as a new hosting vendor. Systems with Restricted data are reassessed every 2 years. | [V] | [Policy 12-025](https://policy.ufl.edu/policy/risk-management-policy/) |
| 12-025 External IT Vendor Sourcing Standard (covers "server hosting … IaaS and 'Cloud' computing"): SLAs are required. Vendors holding Restricted data must sign a **Data Security Agreement** and provide external validation (an audit or certification). And: **"Externally-hosted applications and services used by the university must authenticate users using Gatorlink SSO, unless the primary user base includes those not eligible to obtain Gatorlink accounts."** | [V] | same |
| UFIT Public Cloud: "all UF business with these public cloud vendors [AWS, Azure, GCP] must be done through UFITs managed services". "You are required to complete a risk assessment for every new public cloud project." It needs a UF Hosting account and a UFIT billing number. | [V] | [Public Cloud Services](https://it.ufl.edu/hosting/services/public-cloud-services/), [Public Cloud Support](https://it.ufl.edu/hosting/support/public-cloud-support/) |
| HWCOE IT4-2-0 Website Hosting Requirements: "UFIT service offerings for website hosting must be used whenever possible". "Third-party, external services will not be used for website hosting unless: UFIT service offerings do not provide the required functionality; A UF risk assessment is completed per the UF Integrated Risk Management (IRM) framework." Unit ISMs start the assessments. A lab's primary content counts as "university business". | [V] | [IT4-2-0](https://www.it.eng.ufl.edu/it-administrative-procedures/it4-website-hosting-requirements/), [appendix](https://www.it.eng.ufl.edu/it-administrative-procedures/identifying-websites-used-to-conduct-university-business/) |
| UFIT offers on-prem VM hosting, Postgres hosting, and managed AWS/Azure/GCP. No container service is listed. | [V] | [UFIT Hosting](https://it.ufl.edu/hosting/) |

**What this means [I]:**

- A **VPS bought by the lab** (DigitalOcean, Hetzner, Linode, a personal AWS account) holding Restricted data would need a Data Security Agreement with that vendor, CIO and senior-leadership sign-off, a risk assessment, and a case that UFIT's offerings can't do the job. That is very unlikely to be approved for a small lab app.
- The two realistic routes are (a) a **UFIT-managed AWS/Azure/GCP account** (still off campus, public IP, Docker runs fine; needs a risk assessment and a UFIT billing number) and (b) a **UFIT or HWCOE on-prem VM**. Both start with the same risk assessment.
- **GatorLink SSO is mandatory for externally hosted apps** when the users are GatorLink-eligible, which ours are. Magic-link sign-in becomes at best a fallback that needs an exception, so email would mainly carry reminders. This ties into [#13](https://github.com/uf-cpe-design-lab/stock/issues/13): UF's SP registration also needs a risk assessment, so **one assessment can cover hosting and SSO together**.
- **Data minimization helps, but it doesn't remove the review.** Leaving the course off loans (or making it free text the student enters) might keep the data out of Restricted, but 12-025 still calls for an assessment before acquiring an Information System.

## 5. Who to ask

- **ECE IT:** support@ece.ufl.edu, 352-392-4568, 324 MAE-B. Ask for the service account and alias, the DNS record, CERTInext ACME access, and the name of the unit ISM. [V, [HWCOE IT contacts](https://www.it.eng.ufl.edu/contacts/)]
- **HWCOE IT Director:** it-director@eng.ufl.edu. [V, same]
- **Risk assessment intake:** [security.ufl.edu/submit-a-request](https://security.ufl.edu/submit-a-request/) (behind GatorLink). See also [IRM](https://irm.ufl.edu/). [V that the links exist]
- **DNS:** UFIT IRS (see section 2). **Certificates:** IT-IAMSSLCertAdmin@ufl.edu. [V]
