# Vendor product search / link resolution for Purchase Requests

Research for [#3](https://github.com/uf-cpe-design-lab/stock/issues/3) (map [#1](https://github.com/uf-cpe-design-lab/stock/issues/1)). Checked 2026-10-26.

**Question:** for likely Preferred Vendors, can a Lab User search the vendor or paste a product link and have the Purchase Request pick up part number, title, price, stock and image? Which candidate tools (InvenTree, Snipe-IT, Homebox, Part-DB) already do this?

**Labels:** **V** = verified against the primary source (official docs, the API itself, or project source code). **I** = inferred. **V\*** = official page content seen only through a search-engine extract, because the page timed out for our fetcher.

## Answer

| Vendor | Official API | Search | Link-resolve | Existing integration (tool) | Route for the lab |
|---|---|---|---|---|---|
| **Digi-Key** | Yes, free with registration. OAuth2 2-legged. 1,000 req/day, 120/min | Yes | Yes. Parse the part number from the URL, then call ProductDetails (I) | Part-DB `DigikeyProvider` (search only, no URL handler). InvenTree: third-party plugins/CLI | API |
| **Mouser** | Yes, free API key. 30 calls/min, 1,000/day, 50 results/call (V\*) | Yes | Yes. Parse the MPN from the URL, then call `search/partnumber` (I) | Part-DB `MouserProvider` (no URL handler). InvenTree: supplier-panel, part-import CLI | API |
| **Newark / element14 / Farnell** | Yes, free API key. Limits not on official page; a third party says 2/s and 1,000/day (I) | Yes | Yes: Part-DB has a URL handler | Part-DB `Element14Provider` (search + URL). InvenTree: supplier-panel (Farnell) | API |
| **Adafruit** | Public JSON endpoint, no key (`/api/products`, `/api/product/{id}`) | Yes, by filtering the full catalog | Yes. Product ID is in the URL | Part-DB `AdafruitProvider` (search + URL) | Public JSON |
| **SparkFun** | No official API | Only unofficially | Only unofficially | Part-DB `SparkFunProvider` (search + URL, via the shop's Magento GraphQL plus page scraping, rate-delayed) | Unofficial. Can break |
| **Amazon** | Creators API (PA-API 5 deprecated). Needs Associates enrollment **and 10 qualifying sales in 30 days** | Not usable by the lab in practice | Not usable by the lab in practice. Pasted pages are bot-protected (I) | Part-DB `CanopyProvider` (paid third-party Canopy API) | **Neither.** Store link + manual entry |
| **LCSC** | Official API needs an application and approval (partner-oriented) | Unofficially | Unofficially | Part-DB `LCSCProvider` (unofficial API, search + URL). InvenTree: built-in barcode plugin, part-import CLI | Unofficial |
| **TME** | Yes, token + secret | Yes | Yes: Part-DB has a URL handler | Part-DB `TMEProvider`. InvenTree: barcode plugin, part-import CLI | API (EU-centric vendor) |
| **Arrow** | Pricing & Availability API, needs an authorization key on request | Part-number lookup (V that the API exists; terms not read) | Not checked | None direct. Possibly via Octopart/Nexar, OEMSecrets or TrustedParts aggregators (I) | Aggregator or manual |
| **Octopart / Nexar** (aggregator) | Yes. Free Evaluation plan = 100 matched parts **lifetime**; paid plans 2,000 / 15,000 parts per month | Yes, across distributors | n/a | Part-DB `OctopartProvider` | Too small free tier for routine use |
| **Any other vendor** | n/a | No | Partial: schema.org JSON-LD / OpenGraph on the page, if it isn't bot-blocked | Part-DB `GenericWebProvider` (and an optional LLM-based `AIWebProvider`) | Generic page metadata, then manual |

**Gist:** Digi-Key, Mouser and Newark/element14 have free official APIs that support both search and link-resolve. Adafruit has a free public JSON endpoint. SparkFun and LCSC only work through unofficial routes. Amazon is effectively "neither" for a lab, because API access needs affiliate sales. Among the candidate tools, only Part-DB ships with this (search providers for every vendor above, plus "create part from URL"). InvenTree needs third-party plugins. Snipe-IT and Homebox have nothing usable.

**Terms to watch:** Digi-Key's API agreement forbids using the API "to update or create your own database of information". Mouser's terms reportedly forbid caching or storing content (V\*). Fetching live at request time and storing only the user's chosen part number, URL and quoted price is likely fine. Bulk-populating the inventory catalogue from these APIs is not clearly fine. See open questions.

## Per vendor

### Digi-Key

- **API:** Product Information V4. Endpoints include KeywordSearch (POST), ProductDetails, ProductPricing, Media, Categories, Substitutions, and others (V, [ProductSearch](https://developer.digikey.com/products/product-information-v4/productsearch)).
- **Auth:** OAuth2. The 2-legged flow uses `grant_type=client_credentials` with a 10-minute token. The 3-legged flow uses an authorization code with a 30-minute access token and a non-expiring refresh token. ProductDetails is listed as "Both 2 and 3-legged OAuth" (V, [documentation](https://developer.digikey.com/documentation), [ProductDetails](https://developer.digikey.com/products/product-information-v4/productsearch/productdetails)). 2-legged means a server-side lab tool needs no per-user Digi-Key login (I).
- **Limits:** Product Information is 120/min and 1,000/day on the standard product. Responses include `X-RateLimit-*` headers (V, [documentation](https://developer.digikey.com/documentation)). The agreement says bursting over the per-second threshold triggers "a ten-minute suspension of your account" (V, [API User Agreement §6](https://developer.digikey.com/api-user-agreement)). Higher tiers exist only with manual approval (I, from search results).
- **Cost:** no price stated anywhere on the portal. Free with registration (I; nothing says otherwise).
- **Terms (V, [API User Agreement](https://developer.digikey.com/api-user-agreement)):**
  - §1: an "internal software application to automate and enhance your purchasing of products from DigiKey" is a permitted purpose. A Purchase Request tool fits.
  - §5.1(e) forbids using the API "to update or create your own database of information". §5.1(d) forbids bulk download.
  - §3.1.4 and §5.1(b) require attributing DigiKey as the source and not mixing its data with other data without distinction.
  - §9: inactivity of 90 or more days can suspend the account. On termination you must delete DigiKey Data.
  - The signer must be authorized to bind the entity, so UF procurement or legal may need to accept the terms (I).
- **Link-resolve:** product URLs look like `/en/products/detail/<mfr>/<mfr-part>/<dk-id>`. Extracting the part number and calling ProductDetails should return title, price, stock, photo and URL (I; the response schema wasn't readable without the Swagger download).

### Mouser

- **API:** Search API v1/v2. `POST /api/v1/search/keyword` and `/search/partnumber` each return up to 50 parts. v2 adds `keywordandmanufacturer` and `partnumberandmanufacturer`. Auth is an `apiKey` query parameter (uuid) (V, live Swagger at `https://api.mouser.com/api/docs/V1` and `/V2`). An unauthenticated probe returns `Invalid unique identifier ... API Key` (V).
- **Fields:** the `MouserPart` model includes `MouserPartNumber`, `ManufacturerPartNumber`, `Description`, `ImagePath`, `PriceBreaks`, `Availability`, `AvailabilityInStock`, `ProductDetailUrl`, `DataSheetUrl` and `LifecycleStatus` (V, Swagger). That covers every field a Purchase Request needs.
- **Limits:** up to 50 results per call, 30 calls per minute, 1,000 calls per day (V\*, [Search API page](https://www.mouser.com/en/api-search/)).
- **Terms:** don't exceed assigned call counts, and contact Mouser for more. Reportedly no caching or storing of Mouser content (V\*, [API terms](https://www.mouser.com/en/apiterms/)).
- **Link-resolve:** product URLs are `/ProductDetail/<mfr>/<MPN>?qs=...`. Take the MPN and call `search/partnumber` (I).

### Newark / element14 / Farnell

- **API:** Product Search API (REST). The `term` parameter takes `any:<keyword>`, `id:<element14 PN>` or `manuPartNum:<MPN>`. Response groups `medium`/`large` add prices, stock, images and (v1.3) product URLs. `storeInfo.id` selects the store, including Newark US (V, [description](https://partner.element14.com/docs/Product_Search_API_REST__Description)).
- **Auth:** a 24-character API key. Contract pricing additionally needs a CustomerID and HMAC signature from a sales rep (V).
- **Limits:** not on the official page. A third party cites 2 calls/s and 1,000/day (I).
- **Link-resolve:** Part-DB's `Element14Provider` implements its URL-handler interface (V, via source review of [Part-DB](https://github.com/Part-DB/Part-DB-server/tree/master/src/Services/InfoProviderSystem/Providers)).

### Adafruit

- `GET https://www.adafruit.com/api/product/50` returns JSON with no key: `product_id`, `product_name`, `product_mpn`, `product_price`, `discount_pricing`, `product_stock`, `product_image`, `product_url`, `discontinue_status` (V, live call). `GET /api/products` returns the whole catalogue (about 7.8 MB) (V, live call).
- No published docs or terms were reachable (`/api` returns 403 to our fetcher). Treat it as public but undocumented (I).
- Part-DB's `AdafruitProvider` uses exactly these endpoints and says stock is a number, "in stock", or negative for not orderable (V, [source](https://github.com/Part-DB/Part-DB-server/blob/master/src/Services/InfoProviderSystem/Providers/AdafruitProvider.php)).
- **Link-resolve:** `/product/<id>` maps directly to `/api/product/<id>` (V).

### SparkFun

- There is no official public API. A search found no developer docs, and the old `/products/<id>.json` returns 404 (V).
- Part-DB's `SparkFunProvider` queries the store's public Magento GraphQL endpoint (`https://www.sparkfun.com/graphql`) and scrapes product pages for documents, with a configurable delay between requests (V, [source](https://github.com/Part-DB/Part-DB-server/blob/master/src/Services/InfoProviderSystem/Providers/SparkFunProvider.php)). It works, but it is unofficial and could break when SparkFun changes its store (I).

### Amazon

- PA-API 5 "has been deprecated and is being replaced by the Creators API". PA-API calls now return 403 (V, [deprecation page](https://affiliate-program.amazon.com/creatorsapi/docs/en-us/paapiv5-deprecation)).
- The Creators API (SearchItems, GetItems by ASIN) requires Associates enrollment and "at least 10 qualifying sales within the past 30 days" (V, [introduction](https://affiliate-program.amazon.com/creatorsapi/docs/en-us/introduction)). A lab purchasing tool won't generate affiliate sales, so this route is closed (I).
- Part-DB offers Amazon via Canopy (canopyapi.co), a third-party paid API with a 100 requests/month trial (V per the Part-DB docs; Canopy's own terms not read).
- **Link-resolve:** the ASIN is in the URL (`/dp/<ASIN>`), so the tool can capture a part identifier and link with no API (I). Title, price and image would need scraping, which Amazon blocks and its conditions of use forbid (I). Recommended: store the URL and ASIN, and have the user enter title and price.

### LCSC

- The official API is for partners. You need an LCSC account and an application form that is "evaluated", and requests are signed with key, nonce and timestamp (V\*, [lcsc.com/agent](https://www.lcsc.com/agent), [docs](https://www.lcsc.com/docs/index.html)).
- Part-DB's `LCSCProvider` uses an unofficial endpoint, needs no key, and supports search and URL handling. Its docs call it unofficial and say it may break (V per Part-DB docs/source).

### TME, Arrow, aggregators

- **TME:** official API with token and secret. Part-DB `TMEProvider` has search and a URL handler. InvenTree has a built-in TME barcode plugin (V via source review). TME is EU-based, so it is probably not a primary lab vendor (I).
- **Arrow:** "Access to the Pricing & Availability API requires an authorization key", requested from Arrow. Cost and limits aren't stated (V, [developers.arrow.com](https://developers.arrow.com/api/)). Neither Part-DB nor InvenTree has a direct Arrow integration (V).
- **Nexar / Octopart:** Evaluation plan is up to 100 matched parts, free. Standard is up to 2,000/month and Pro up to 15,000/month; prices aren't listed (V, [compare plans](https://nexar.com/compare-plans)). Support docs say the evaluation limit is lifetime and does not reset (V\*, [part limits](https://support.nexar.com/support/solutions/articles/101000476314-part-limits-and-how-they-work)). It is useful only as a paid fallback.

## Candidate tools

### Part-DB (v2.20.3): built-in

- `src/Services/InfoProviderSystem/Providers/` includes Digikey, Mouser, Element14, TME, LCSC, Octopart, Adafruit, SparkFun, Canopy (Amazon), OEMSecrets, TrustedParts, GenericWeb, AIWeb, AIDocument, Reichelt, Pollin, Pololu, Buerklin, Conrad and BambuLab (V, [directory listing](https://github.com/Part-DB/Part-DB-server/tree/master/src/Services/InfoProviderSystem/Providers)).
- **Create from URL:** route `/tools/info_providers/from_url`, gated by the `info_providers.create_parts` permission. It tries the provider that owns the URL's domain (Element14, TME, LCSC, Adafruit, SparkFun, ...) and otherwise falls back to `GenericWebProvider`. That provider reads schema.org `Product` JSON-LD and then OpenGraph `og:title`/`og:image` (V, source via subagent review). A browser plugin can submit an already-loaded page to get past bot checks (V per docs).
- Search and detail results are cached for 7 and 4 days (V per [docs](https://docs.part-db.de/usage/information_provider_system.html)). That cache may sit awkwardly with the Digi-Key and Mouser no-database clauses (I).
- **Caveat:** Part-DB's flow creates a **Part** in the catalogue. It is not a Purchase Request. Whether Part-DB has any request or ordering workflow is a separate question (I; not covered here).

### InvenTree (1.6.1): plugin route

- Built-in `digikey.py`, `mouser.py`, `lcsc.py` and `tme.py` are `SupplierBarcodeMixin` plugins. They decode labels on received packaging; they don't search (V, source).
- `SupplierMixin` plus the part-import wizard (since 1.1.0) provides the framework: search supplier, pick category, map parameters, create stock. Core ships only a dummy sample (V, [docs](https://docs.inventree.org/en/latest/plugins/mixins/supplier/)).
- Real supplier search comes from third-party code: `inventree-supplier-panel` (Mouser/DigiKey/Farnell; listed on inventree.org/plugins), `inventree-part-import` (a CLI for DigiKey/Mouser/LCSC/TME/Reichelt), and a few small `SupplierMixin` plugins (V that they exist; quality not reviewed).
- Purchase Orders exist (Pending to Placed to Complete) with role permissions. There is no request or approval workflow, but a Pending PO created by a non-staff user with PO-add rights could stand in for a request (I).

### Snipe-IT (v8.8.3): none

Suppliers are plain contact records. A code search for vendor names finds nothing. The new `Order` model is explicitly "NOT a purchase-order workflow" (V, source).

### Homebox (sysadminsmedia, v0.27.1): consumer barcodes only

`GET /v1/products/search-from-barcode` queries UPCitemdb, BarcodeSpider (optional token) and Open*Facts by UPC/EAN (V, source). It has no distributor integration and no URL import (V/I).

## Open questions for the map

1. **Terms vs. storing data.** Does saving Digi-Key or Mouser API results into the inventory catalogue (Part records, cached prices) breach Digi-Key §5.1(e) or Mouser's no-caching clause? Fetching live inside a Purchase Request and storing only what the user chose is lower risk. Who at UF accepts these API terms (the signer must be able to bind the university)?
2. **Purchase Request home.** Part-DB has the best vendor lookup but creates Parts, not requests. InvenTree has Purchase Orders but needs plugins for lookup. Does the Purchase Request live in the inventory tool, or in a thin front end that calls the vendor APIs?
3. **Which vendors are actually Preferred?** If the list is Digi-Key, Mouser, Adafruit, SparkFun and Amazon, then Amazon (and arguably SparkFun) will always need a link plus manual entry. Confirm the list, and whether Newark is on it.
