# How does GatorLink sign-in work for a lab-hosted app?

Research for [#13](https://github.com/uf-cpe-design-lab/stock/issues/13) (map [#1](https://github.com/uf-cpe-design-lab/stock/issues/1)). Checked 2026-10-26 against primary sources: UFIT Identity & Access Management (IAM) pages on it.ufl.edu, the live UF IdP metadata and OIDC discovery documents on login.ufl.edu, and each candidate tool's own docs and source code. Domain terms follow `GLOSSARY.md`.

**Tags:** **[V]** means verified at the linked source. **[I]** means inferred from the source, from the SAML/OIDC specs, or from the absence of something, and not tested against UF's IdP. Nothing was registered with UF, and no tool was installed.

**What could not be seen:** the SP Registry (`admin.ict.it.ufl.edu/sp/sp.cgi`), the metadata upload page and the IAM TeamDynamix (TDX) ticket forms all redirect to GatorLink login. Their fields, and any rules shown only after login (for example who may be listed as a contact), were not read.

## Answer in one paragraph

UF runs one Shibboleth IdP (`login.ufl.edu`) that speaks **SAML 2.0 and OpenID Connect**. No CAS or LDAP sign-in for department apps is documented. To use it, someone who will be a **technical or security contact** for the app registers a *Service Provider* (SP) in UF's SP Registry. The steps: complete a UF Information Security **risk assessment** first (with the unit's ISM), make sure the department knows about the request, name at least two contacts, and **recertify every year**. IAM reviews and approves the request. A SAML app then uploads its own metadata; an OIDC app collects a client ID and secret from the Registry. Changes reach the beta IdP the same afternoon and production the next morning. Attributes are released on a minimum-need basis. The documented ones are `glid`, `mail`, `primary-affiliation` (STUDENT, STAFF, FACULTY, GUEST) and `UFAD_Groups`. UF's OIDC discovery lists only the standard profile and email claims, so **SAML is the route if the app needs affiliation or groups**. Course rosters are not offered as an SSO attribute anywhere UF documents. **The app does not have to be reachable from the internet.** Both protocols pass through the user's browser, and UF takes the SP's metadata by upload, not by fetching it. The server only needs outbound HTTPS to `login.ufl.edu`, a stable hostname that users' browsers can reach, and (strongly advised) a real TLS certificate. Unit IT can issue a free `ufl.edu` certificate. Directory (UFAD) access for building accounts ahead of time is not documented for department apps, so plan on **creating accounts when a person first signs in** (just in time). Of the candidates, **InvenTree 1.6+ fits best** (SAML or OIDC, self-registration through SSO, group sync). **Part-DB** works but its SAML is beta. **Shelf.nu** self-hosted needs Supabase's SAML plus group mapping. **Snipe-IT** is the weakest: SAML can only sign in accounts that already exist, and its usual fix (LDAP sync) is not available to us.

## 1. Protocols UF offers

| Protocol | Status at UF | Evidence |
|---|---|---|
| SAML 2.0 (Shibboleth) | **Offered**, preferred for vendor apps | "UF's implementation supports SAML2 and OpenID Connect (OIDC) protocols." Vendors: "SAML2 is preferred, OIDC as fallback." [V, [SSO page](https://it.ufl.edu/iam/authentication-systems/single-sign-on/)] |
| OpenID Connect | **Offered** | Same page; live discovery doc at [`login.ufl.edu/.well-known/openid-configuration`](https://login.ufl.edu/.well-known/openid-configuration) [V] |
| CAS | Not documented | No UF page offers it. The Shibboleth IdP software answers on `/idp/profile/cas/login`, but that only shows the software has the endpoint, not that UF will register a CAS client. Treat as unavailable. [I] |
| LDAP sign-in (bind to UFAD) | Not documented for apps | UFAD is "the on-premises domain service used for Gatorlink account authentication" ([UFIT service 505](https://it.ufl.edu/services/505)), but no page describes app binds or service accounts. `ad.ufl.edu` resolves to private `10.241.x.x` addresses (campus-internal). The SSO policy says "All SPs must use an approved SSO method", with exceptions needing IAM and Security Office sign-off. [V for quotes and DNS; I that LDAP is effectively off the table] |
| InCommon federation | Not needed | The InCommon page covers letting **non-UF** people sign in to UF apps (Research & Scholarship category, REFEDS MFA). An SP that uploads metadata to UF directly does not need InCommon. [V, [Federated Authentication](https://it.ufl.edu/iam/authentication-systems/incommon-federated-authentication/); I on "not needed"] |
| Social / UF Guest Login (Cirrus) | Exists, "very limited capacity" | Lets non-GatorLink people sign in with Google/Microsoft/LinkedIn or a self-service UF Guest account. How an app gets onto Cirrus is not described. [V, [Social and UF Guest Login](https://it.ufl.edu/iam/authentication-systems/social-and-uf-guest-login/)] |

IdP endpoints [V]:

- SAML metadata: production `https://login.ufl.edu/login.ufl.edu.xml` (entityID `https://login.ufl.edu/idp/shibboleth`), beta `https://beta.login.ufl.edu/beta.login.ufl.edu.xml`. Supports HTTP-Redirect and HTTP-POST bindings.
- OIDC: issuer `https://login.ufl.edu`. Supports authorization-code flow, `client_secret_basic` / `client_secret_post`, and scopes `openid profile email address phone offline_access`. Beta: `https://beta.login.ufl.edu/.well-known/openid-configuration`. The discovery doc advertises a dynamic `registration_endpoint`, but UF's process issues the client ID and secret through the SP Registry. Don't rely on self-registration. [V for the doc, I on not relying on it]

## 2. Registration process and who can request

From the [SSO page](https://it.ufl.edu/iam/authentication-systems/single-sign-on/) and its [SP Request Workflow PDF](https://it.ufl.edu/iam/media/itufledu/iam/UF-SP-Request-Workflow.pdf) (updated 2025-01-03) [V]:

1. **Risk assessment first.** "All new SPs (except for WordPress sites) are required to have an up-to-date risk assessment." You need the assessment number to request the SP, and the SP is not approved until the assessment is complete. Submit it at [security.ufl.edu/submit-a-request](https://security.ufl.edu/submit-a-request/); the unit's **ISM** (Information Security Manager) runs it ([Risk Assessment page](https://it.ufl.edu/security/audiences/faculty--staff/risk-assessment/)). No timeline is published. The only exemption is a "Fast Path" solution. Fast Path criteria include "Solution provides native SSO support" and "supports multiple use-cases such as administration, research, and teaching", and the list is curated by UFIT. A lab-hosted tool would not be on it. [V for criteria, I that we don't qualify]
2. **Request the SP** in the [SP Registry](https://admin.ict.it.ufl.edu/sp/sp.cgi) via "Request New SP Group". *(Behind GatorLink, not seen.)*
3. **Who:** "requestor **must be** one of the technical or security contacts for the SP." Each SP needs **at least two unique contacts**, verified yearly. Policy also asks for "departmental awareness". UF does not say whether a student may be a contact. **Recommendation [I]:** list a faculty or staff lab owner and the department/college IT contact as the two contacts, and have the ISM involved from step 1.
4. **IAM review.** An IAM member is assigned. If the risk assessment is incomplete, the request is rejected unless the Security Office has approved "the SSO implementation in its current state". IAM may "work with the requesting department" and adjust the request.
5. **On approval:** for **SAML**, the SP owner uploads SP metadata at the [metadata upload page](https://admin.ict.it.ufl.edu/shibboleth/upload-metadata.cgi). For **OIDC**, the owner "retrieves secret id and secret key from SP registry".
6. **Deployment cadence:** metadata goes to **beta login** daily at 2 PM (done by 3:30 PM), then to **production** next day at 7 AM (done by 8 AM). Expect roughly one business day from approval to production.
7. **Ongoing duties:** recertify the SP group **yearly** (overdue SPs may be disabled). SPs inactive for **6 months are deactivated** (reactivated on request). Keep signing certificates current, and request only the attributes you need. Significant changes may need a new risk assessment.

Testing: UF recommends a test service and separate entityIDs per environment. Point a dev instance at `beta.login.ufl.edu` with its own entityID. [V for the recommendation, I for how we apply it]

Contact: `Identity-services@it.ufl.edu`, or an IAM TDX ticket. [V]

## 3. Attributes UF releases

UF publishes **no attribute catalogue or release policy page** that could be found. What can be pinned down [V, SSO page]:

- Policy: "The minimum required attributes should be used in order to perform the necessary function of the SP." You list the attributes or claims you need when requesting.
- Attributes named in UF's own access-control examples: **`glid`** (GatorLink username), **`mail`**, **`primary-affiliation`** (values shown: `STAFF`, `FACULTY`, `STUDENT`, `GUEST`), **`UFAD_Groups`** (semicolon-separated UFAD group names).
- NameID: "if the vendor has specific needs for the NameID, we need to know what they are." So a specific NameID (persistent, or the GatorLink ID) can be requested.
- OIDC `claims_supported` lists only standard claims: `sub, email, name, given_name, family_name, middle_name, profile, address, phone, …`. No affiliation or groups claim is advertised. [V, discovery doc]

Not found anywhere [I]:

- `eduPersonPrincipalName`, `displayName`, `eduPersonAffiliation`. These are standard Shibboleth attributes and probably exist (InCommon R&S requires ePPN), but UF never names them for internal SPs. Ask IAM.
- **Course rosters or course membership.** Nothing in UF's SSO docs offers them. Whether UFAD has per-course groups that `UFAD_Groups` would carry is unknown, and they would probably be heavy to release. Rosters live in Canvas or the registrar systems, not SSO. Course and project on a Loan should stay user-entered, as the map already assumes.
- **Lab-specific groups.** A "CPE Lab Staff" UFAD group would need a departmental UFAD admin to create it. Without one, Lab Staff vs. Lab User is assigned **inside the app**.

## 4. Directory/LDAP: pre-provision or just-in-time?

- No UF page offers department apps LDAP binds, a directory service account, or SCIM provisioning from UF. [V, by absence across IAM pages]
- The public UF directory (`directory.ufl.edu`) is a web phonebook, not a provisioning feed. [V that it exists; I on its use]
- **Conclusion [I]:** plan for **just-in-time accounts**, created the first time someone signs in with GatorLink. Gate who gets in with `primary-affiliation` or an email domain, give everyone the Lab User role by default, and have Lab Staff promote staff by hand. A semester roster import (CSV from the instructor) is the fallback for tools that cannot do just-in-time. Asking IAM for an LDAP service account is possible but needs a policy exception ("must use an approved SSO method"), so it is unlikely to be worth it.

## 5. Network, hostname and TLS requirements

| Need | Answer | Basis |
|---|---|---|
| Reachable from the internet? | **No.** SAML Web Browser SSO (Redirect/POST bindings) and the OIDC authorization-code flow both move through the user's browser. The IdP never connects to the SP, and UF takes SP metadata by **upload**, not by fetching it. | [V for upload-based metadata; I from SAML/OIDC specs]. InvenTree's docs say so directly: "Your server does not need to be 'public facing' for this to work." [V] |
| Outbound access | The server must reach `https://login.ufl.edu`: OIDC token and userinfo calls go back-channel, and some SAML libraries fetch IdP metadata by URL. | [I from OIDC spec and tool configs] |
| Stable hostname | **Yes.** The ACS URL (SAML) or redirect URI (OIDC) is registered with UF and must exactly match what users' browsers load. Changing it means a Registry change, a new upload and another one-day deploy. A DNS name (ideally under `ufl.edu`) is needed, not a bare IP. This overlaps the map's open "Hostname & phone access" item. | [I] |
| TLS certificate | Strongly advised, and in practice required. UF's own Windows SP prerequisites list "Certificate bound to 443/tcp", and allauth, Supabase and Part-DB all expect HTTPS callbacks. Free `ufl.edu` certificates come through the college or unit IT office ("UFIT IAM doesn't directly provide SSL Certificates"), now issued by CERTInext under InCommon, with *.ufl.edu exempt from per-domain validation. Lifetimes are shrinking: 200 days now, 100 days in 2027, 47 days in 2029, so automate renewal. | [V, [SSL Certificates](https://it.ufl.edu/iam/authentication-systems/ssl-certificates/); I that UF's IdP would reject `http` ACS URLs, which is untested] |
| SAML signing certificate | **Required** ("using signing certificates; encryption certificates are desirable"). This is the SP's own key pair in its metadata and can be self-generated. It is separate from the HTTPS certificate. Do not let it expire. | [V policy; I that self-signed is accepted, which is standard Shibboleth practice] |
| entityID | Any unique URI you choose (e.g. `https://stock.<dept>.ufl.edu/saml/metadata`). It does not have to resolve. | [I, SAML spec] |
| MFA | Enforced by UF's IdP at login. Nothing for the app to do. | [I] |

## 6. Fit per candidate tool

| Tool | GatorLink path | Just-in-time accounts | Roles from UF | Verdict |
|---|---|---|---|---|
| **InvenTree 1.6.x** | **SAML or OIDC.** 1.6.0 enables allauth's `saml` and `openid_connect` providers by default and adds "Add SAML App" / OIDC forms in the Admin Center, which show the ACS, SLS and metadata URLs `/accounts/saml/<org>/acs/` etc. ([PR #12799](https://github.com/inventree/InvenTree/pull/12799), [PR #12731](https://github.com/inventree/InvenTree/pull/12731), [1.6.0 release](https://github.com/inventree/InvenTree/releases/tag/1.6.0)). LDAP is also supported (`django-auth-ldap`), and so is `REMOTE_USER` header login. [V] | **Yes.** `LOGIN_ENABLE_SSO_REG` (self-registration via SSO) + `LOGIN_SIGNUP_SSO_AUTO` (fill details from SSO). Email must be configured first. Can restrict signup with `LOGIN_SIGNUP_MAIL_RESTRICTION=@ufl.edu`. [V, `common/setting/system.py` @1.6.1, [SSO docs](https://docs.inventree.org/en/latest/settings/SSO/)] | Optional group sync (`SSO_GROUP_KEY`, `SSO_GROUP_MAP`) could map `UFAD_Groups` if UF releases it. Tested upstream only with Keycloak OIDC. [V/I] | **Best fit.** Use OIDC for a simple start (no affiliation available), or SAML if affiliation/group gating is wanted. |
| **Part-DB** | SAML only, **beta** ("only been tested with Keycloak"). Uses OneLogin php-saml with signed AuthnRequests (`authnRequestsSigned: true`). [V, [saml_sso.md](https://github.com/Part-DB/Part-DB-server/blob/master/docs/installation/saml_sso.md), `config/packages/nbgrp_onelogin_saml.yaml`] | **Yes.** "the user is created on the first login". [V] | `SAML_ROLE_MAPPING` reads a SAML attribute literally named **`group`**. UF would have to release `UFAD_Groups` under that name, which is unlikely, so use `"*"` → default group and promote by hand. **Username = SAML NameID**, so ask IAM for a NameID equal to the GatorLink ID (stable). Email/name attributes expected as `email`/`firstName`/`lastName` or OIDs. [V for config; I for UF side] | Workable. Beta status and the NameID ask add risk. |
| **Shelf.nu (self-hosted)** | SAML only, through **Supabase Auth** (`GOTRUE_SAML_ENABLED`, providers added at runtime through the admin API). The SP entityID and ACS hang off `API_EXTERNAL_URL/sso/saml/…`. Supabase docs say this must be "the publicly-accessible URL" the IdP can reach, but under SAML POST binding only the user's browser posts to it (reachable on the LAN is enough [I]). [V, [Supabase self-hosted SAML](https://supabase.com/docs/guides/self-hosting/self-hosted-saml-sso.md)] | **Yes**, on first SSO sign-in. Needs a **persistent NameID** ("transient" is rejected) and `mail`, `givenName`, `sn`. Requires a non-SSO owner account created **before** the domain is set to SSO. Once `ufl.edu` is an SSO domain, password logins on it stop. [V, Shelf [sso/index.md](https://github.com/Shelf-nu/shelf.nu/blob/main/apps/docs/sso/index.md), [providers/shibboleth.md](https://github.com/Shelf-nu/shelf.nu/blob/main/apps/docs/sso/providers/shibboleth.md)] | **Mandatory group mapping:** roles come only from a released group attribute (`isMemberOf`, `eduPersonEntitlement` or `eduPersonScopedAffiliation`). Users matching no group sit on a "pending assignment" screen. UF's documented attributes are `primary-affiliation`/`UFAD_Groups`, so IAM would have to release one of those under a name Shelf reads. [V for Shelf; I for UF] | Possible but the most moving parts. Hosted Shelf SSO is a paid plan, and self-hosted relies on Supabase's SAML admin API. |
| **Snipe-IT** | SAML. "SAML can only authenticate a user in Snipe-IT if the user exists in Snipe-IT"; recommends "LDAP sync or SCIM to pull your users in, then SAML". Can match on the ePPN OID `urn:oid:1.3.6.1.4.1.5923.1.1.1.6` instead of NameID. [V, [Snipe-IT SAML docs](https://snipe-it.readme.io/docs/saml)] | **No.** LDAP sync needs UFAD bind access (not available, §4), and UF offers no SCIM feed. Accounts would come from a **CSV import each semester** before students can sign in. [I] | Set by hand in Snipe-IT. | **Weakest fit** for students: walk-up Lab Users without a pre-imported account cannot sign in. |

**Fallback for any tool [I]:** put a reverse proxy in front that handles UF sign-in (Apache `mod_shib`, or `oauth2-proxy` with UF OIDC) and passes the GatorLink ID in a header. InvenTree supports this natively (`remote_login_enabled` / `remote_login_header`). It still needs the same SP registration.

## 7. Suggested path (inferred, for the spec)

1. Pick the tool (#2/#10). If it is InvenTree, start with **OIDC** (simplest: client ID and secret, `openid email profile`). Move to SAML only if gating by `primary-affiliation` or `UFAD_Groups` is wanted.
2. Get a DNS name and a `ufl.edu` TLS certificate through department/college IT.
3. File the risk assessment with the unit ISM. Describe the data: names, emails, GatorLink IDs and Loan history tied to courses (possible FERPA angle, so ask the ISM).
4. Request two SPs in the Registry (prod + test on `beta.login.ufl.edu`), with a faculty/staff owner and department IT as contacts.
5. Configure just-in-time accounts restricted to `@ufl.edu`, defaulting to Lab User. Lab Staff are promoted in the app.

## Open questions for UF IAM / department IT

- May a student be listed as an SP technical contact, or must both contacts be faculty/staff?
- Exact attribute names releasable to a department SP: ePPN, `displayName`, `primary-affiliation`, `UFAD_Groups`? Can OIDC carry affiliation or groups as custom claims?
- Does the IdP accept `http://` or private-IP ACS/redirect URLs? (Assume no.)
- Which risk-assessment tier applies to a lab inventory app holding student names and emails, and how long does it take?
- Can the department create a UFAD group for Lab Staff that is released via `UFAD_Groups`?

## Sources

- UFIT IAM, Single Sign On: https://it.ufl.edu/iam/authentication-systems/single-sign-on/
- UFIT IAM, SP Request Workflow (PDF, 2025-01-03): https://it.ufl.edu/iam/media/itufledu/iam/UF-SP-Request-Workflow.pdf
- UFIT IAM, Authentication Systems index: https://it.ufl.edu/iam/authentication-systems
- UFIT IAM, Federated Authentication (InCommon): https://it.ufl.edu/iam/authentication-systems/incommon-federated-authentication/
- UFIT IAM, SSL Certificates: https://it.ufl.edu/iam/authentication-systems/ssl-certificates/
- UFIT IAM, Social and UF Guest Login: https://it.ufl.edu/iam/authentication-systems/social-and-uf-guest-login/
- UFIT IAM, FAQs: https://it.ufl.edu/iam/faqs/
- UFIT, Active Directory service: https://it.ufl.edu/services/505
- UF Information Security, Risk Assessment: https://it.ufl.edu/security/audiences/faculty--staff/risk-assessment/
- UF Information Security, Fast Path criteria: https://it.ufl.edu/security/audiences/faculty--staff/fast-path-solutions/fast-path-solutions-criteria/
- UF IdP SAML metadata: https://login.ufl.edu/login.ufl.edu.xml
- UF IdP OIDC discovery: https://login.ufl.edu/.well-known/openid-configuration
- InvenTree SSO docs: https://docs.inventree.org/en/latest/settings/SSO/; 1.6.0 release notes: https://github.com/inventree/InvenTree/releases/tag/1.6.0; PR #12799: https://github.com/inventree/InvenTree/pull/12799
- django-allauth OpenID Connect provider: https://docs.allauth.org/en/latest/socialaccount/providers/openid_connect.html
- Snipe-IT SAML: https://snipe-it.readme.io/docs/saml
- Shelf.nu SSO docs: https://github.com/Shelf-nu/shelf.nu/tree/main/apps/docs/sso
- Supabase self-hosted SAML: https://supabase.com/docs/guides/self-hosting/self-hosted-saml-sso.md
- Part-DB SAML: https://github.com/Part-DB/Part-DB-server/blob/master/docs/installation/saml_sso.md
