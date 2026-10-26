# Printing QR labels on the Brother QL-820NWB

Research for issue #4 (child of the wayfinder map, #1). Researched 2026-10-26.

**Legend.** **[V]** means verified: read in a primary source (Brother spec, catalog or developer manual, project source code or README, QR spec owner, or scanner datasheet) during this research. **[I]** means inferred: derived from verified facts, computed, or general practice that wasn't confirmed in a primary source. Run a print-and-scan test before you rely on any [I] item.

## TL;DR

- **Print route:** a Linux Docker host can send raster jobs straight to the printer's raw TCP port (`tcp://<printer-ip>:9100`) with the **`brother_ql_next`** Python library (maintained, Python 3.9+, QL-820NWB verified). This needs no CUPS, no Brother driver and no Windows. [V]
- **Tool support:** **InvenTree** is the only candidate that prints to the QL-820NWB natively, through the official `inventree-brother-plugin`. **Snipe-IT** and **Part-DB** produce PDFs that you print through the browser and the OS print dialog. **Homebox** produces PNGs and can run a configurable print command or call an external label service. [V]
- **Stocks:** use die-cut **DK-1204 (17×54 mm)** or **DK-1209 (29×62 mm)** for bins and drawers, **DK-2251 (62 mm continuous, red/black)** cut to length for cabinets and locations, and **DK-2212 (62 mm continuous film)** or **DK-1201 (29×90 mm paper)** for Equipment asset tags. [V for the stock facts; the per-use choices are a recommendation]
- **QR floor:** use a module of at least 4 printer dots (0.34 mm) and QR version 3 with error-correction level M. That holds a URL of up to 42 bytes in a **9.8 mm symbol, or 12.5 mm with the quiet zone**. Use 5 to 6 dots (0.42 to 0.51 mm) where the label has room. A typical 2D imager such as the Zebra DS2208 is rated down to a 6.7 mil (0.17 mm) QR module, so the phone camera is the tighter limit. [V for the spec and datasheet figures; I for the floor]

---

## 1. The printer, briefly

| Fact | Value | Source |
|---|---|---|
| Print method | Direct thermal (no ink or ribbon; labels are heat-sensitive) | [V] Brother spec page |
| Resolution | 300×300 dpi standard, 300×600 dpi high-res mode (600 dpi in feed direction only) | [V] Brother spec page; Raster Command Reference p.31 (`ESC i K` bit 7) |
| Max print width | 58 mm (62 mm media, 58.9 mm / 696 dots printable) | [V] spec page; Raster ref §2.3 |
| Continuous label length | 12.7 mm min to 1000 mm max | [V] Raster ref §2.3.3 |
| Interfaces | Wi-Fi (infrastructure + Wireless Direct), 10/100 Ethernet, USB 2.0, Bluetooth Classic, plus a USB host port (HID class, so a keyboard or scanner can plug into the printer) | [V] spec page |
| Two-color | Red+black, **only on DK-2251 (EU: DK-22251)** | [V] Brother DK-2251 page; brother_ql README |
| Command modes | ESC/P (default), Raster, P-touch Template (switch with `ESC i a n`) | [V] Raster ref p.29 |

## 2. DK label stocks

All rolls below are listed by Brother as QL-820NWB consumables. US part numbers are shown; the EU/UK equivalent adds a digit (DK-1201 → DK-11201, DK-2205 → DK-22205, DK-2251 → DK-22251). [V for the US and UK consumables lists; the DK-2251 → DK-22251 pairing is I, by the same pattern]

"Printable" is the area Brother's raster reference gives for each size, at 300 dpi. [V, Raster ref §2.3.2]

| Part (US / EU) | Type | Size (mm) | Printable (mm / dots) | Per roll | brother_ql id |
|---|---|---|---|---|---|
| DK-1219 / DK-11219 | die-cut round, paper | Ø12 | Ø8.0 / 94 | 1200 | `d12` |
| DK-1204 / DK-11204 | die-cut, paper | 17×54 | 14.0×47.9 / 165×566 | 400 | `17x54` |
| DK-1203 / DK-11203 | die-cut, paper | 17×87 | 14.0×80.9 / 165×956 | 300 | `17x87` |
| DK-1221 / DK-11221 | die-cut square, paper | 23×23 | 20.0×17.1 / 236×202 | 1000 | `23x23` |
| DK-1218 / DK-11218 | die-cut round, paper | Ø24 | Ø20.0 / 236 | 1000 | `d24` |
| DK-3235 | die-cut, **removable** paper | 29×54 | 22.9×51.0 / 271×602 | 800 | `54x29` |
| DK-1209 / DK-11209 | die-cut, paper | 29×62 | 22.9×58.9 / 271×696 | 800 | `62x29` |
| DK-1201 / DK-11201 | die-cut, paper | 29×90 | 25.9×83.9 / 306×991 | 400 | `29x90` |
| DK-1208 / DK-11208 | die-cut, paper | 38×90 | 35.0×83.9 / 413×991 | 400 | `39x90` |
| DK-1202 / DK-11202 | die-cut, paper | 62×100 | 58.9×93.9 / 696×1109 | 300 | `62x100` |
| DK-2210 / DK-22210 | continuous, paper | 29 wide | 25.9 / 306 wide | 30.48 m | `29` |
| DK-2211 / DK-22211 | continuous, **film** | 29 wide | 25.9 / 306 wide | 15.24 m | `29` |
| DK-2205 / DK-22205 | continuous, paper | 62 wide | 58.9 / 696 wide | 30.48 m | `62` |
| DK-2212 / DK-22212 | continuous, **film**, white | 62 wide | 58.9 / 696 wide | 15.24 m | `62` |
| DK-4205 / DK-44205 | continuous, **removable** paper | 62 wide | 58.9 / 696 wide | 30.48 m | `62` |
| **DK-2251 / DK-22251** | continuous, paper, **black+red on white** | 62 wide | 58.9 / 696 wide | 15.2 m | `62red` |

Notes:

- **Red/black is a 62 mm continuous paper roll only.** No die-cut or film two-color stock exists, and you set the length per label. [V, Brother DK-2251 page and consumables list] With brother_ql you must pass `--red` (label `62red`) whenever this roll is loaded, "even when not printing red". [V, brother_ql README]
- **Paper vs. film.** Brother's DK-2212 page makes no durability claims. [V] Film is generally more resistant to handling, water and tearing than direct-thermal paper, and both fade with heat and UV. [I] Asset tags on Equipment that gets handled and loaned out should use film, or paper under a clear overlay. [I]
- **Custom die-cut sizes.** Brother's raster reference also lists 62×60 and 62×75 die-cut sizes that brother_ql_next's label table doesn't have. [V] Stick to the sizes in the table above.

### Recommended stock per use [I, a recommendation built on the [V] dimensions]

| Use | Primary | Alternative | Why |
|---|---|---|---|
| Small-parts **drawers** (narrow drawer fronts) | **DK-1204** 17×54 die-cut | DK-1221 23×23 (QR + short ID only) | Fits a typical drawer front. QR v3 at 4-dot modules plus 2–3 short text lines fit (see §3.3). |
| **Bins** (larger front faces) | **DK-1209** 29×62 die-cut | DK-3235 29×54 removable, if bins get re-slotted | Room for a 15–20 mm QR plus name, part number and location. |
| **Cabinets / shelves / Locations** | **DK-2251** 62 mm red/black, cut ~62×30–50 | DK-2205 62 mm black | A large red location code reads from across the room. Cut length per label. |
| **Equipment asset tags** | **DK-2212** 62 mm film, cut ~62×25 (or DK-2211 29 mm film) | DK-1201 29×90 paper (cheaper) | Durable, and the same roll can make different lengths. |

One die-cut roll and one continuous roll can be loaded at a time, and the printer holds only one roll. Changing rolls between print batches is part of the workflow. [I] The roll-change cost is a reason to use few stocks: for example **DK-1209 for all bins and drawers, DK-2251 for locations, and DK-2212 for Equipment**.

## 3. QR size floor and what fits beside it

### 3.1 Spec facts [V]

- **Symbol size:** version *n* is (17 + 4n) modules square: v1 21×21, v2 25×25, v3 29×29, v4 33×33. [V, DENSO WAVE "Information capacity and versions"]
- **Quiet zone:** "QR Code requires a four-module wide margin at all sides of a symbol." [V, DENSO WAVE "Point for determining the code area"]
- **Capacity at error-correction level M** (the level DENSO says is most often chosen): [V, DENSO WAVE version table]

  | Version | Binary bytes (L / **M** / Q) | Alphanumeric (L / **M**) |
  |---|---|---|
  | v2 (25×25) | 32 / **26** / 20 | 47 / **38** |
  | v3 (29×29) | 53 / **42** / 32 | 77 / **61** |
  | v4 (33×33) | 78 / **62** / 46 | 114 / **90** |

  A typical item URL such as `https://stock.cpe.ufl.edu/part/1234/` is about 35 bytes, so it needs **v3-M**. A shorter host and path, such as `http://inv.lan/p/1234`, fits **v2-M**. [I] Uppercase alphanumeric mode packs more characters, but the item tools' URL paths are case-sensitive, so don't plan on it. [I]
- **Handheld 2D imager example:** the Zebra DS2208 (DS2200 series) has a minimum element resolution for QR of **6.7 mil (0.17 mm)**. Its decode range for a 20 mil QR is 0–27.9 cm, and its sensor is 640×480. [V, Zebra DS2200 spec sheet] Honeywell's 1470g datasheet wasn't reachable. Reseller listings give 5 mil for linear codes but no QR module figure. [not verified]

### 3.2 Size at 300 dpi

One printer dot is 25.4 / 300 = 0.0847 mm. Use a **whole number of dots per module**; scaling a QR by a fraction smears modules on a thermal head. The InvenTree plugin and brother_ql both resize the image, so render the QR at the exact final pixel size. [I]

| Dots/module | Module | v2 symbol / with QZ | v3 symbol / with QZ | v4 symbol / with QZ |
|---|---|---|---|---|
| 3 | 0.254 mm (10 mil) | 6.4 / 8.4 mm | 7.4 / 9.4 mm | 8.4 / 10.4 mm |
| **4** | **0.339 mm (13.3 mil)** | 8.5 / 11.2 mm | **9.8 / 12.5 mm** | 11.2 / 13.9 mm |
| 5 | 0.423 mm (16.7 mil) | 10.6 / 14.0 mm | 12.3 / 15.7 mm | 14.0 / 17.4 mm |
| 6 | 0.508 mm (20 mil) | 12.7 / 16.8 mm | 14.7 / 18.8 mm | 16.8 / 20.8 mm |

[I: arithmetic from the [V] module counts and 300 dpi]

**Recommended floor [I]:**

- **USB handheld 2D imager:** 3 dots (0.25 mm) is already 1.5× the DS2208's 6.7 mil minimum. The scanner is not the limit.
- **Phone camera:** use **≥ 4 dots (0.34 mm) and a symbol ≥ ~10 mm**. Prefer 5–6 dots (≥ 12 mm) wherever the label allows. Phones need to focus close, often 7–10 cm, to resolve smaller codes, and older or budget phones and cluttered drawer fronts fail first. No phone vendor publishes a minimum module size, so **this floor needs a print-and-scan test with the lab's actual phones** before you commit to the 17 mm stock.
- **Quiet zone:** the unprinted 1.5 mm edge of a die-cut label counts as white margin. On small labels you can butt the QR against the printable edge and let the label edge supply part of the quiet zone. Don't put text within 4 modules of the QR on the other sides. [I, built on the [V] 1.5 mm margins in the Raster ref]
- **1D codes:** a 1D code isn't needed for USB scanning if the lab buys 2D imagers. A 1D-only laser scanner can't read QR. [I] Buy 2D imagers.

### 3.3 Text beside the QR (300 dpi; 1 pt ≈ 4.17 dots) [I, computed]

Assumes a condensed sans font, about 0.5 em average character width and 1.2× line spacing. A 7 pt font is the practical legibility floor on direct thermal.

| Stock | QR | Text area left | Fits |
|---|---|---|---|
| DK-1204 17×54 (165×566 dots) | v3 at 4 dots, ~10 mm | ~35×14 mm | 3 lines at 8 pt (~25 chars each) or 2 lines at 11 pt (~18 chars): e.g. name, value or package, and location code |
| DK-1221 23×23 (236×202 dots) | v3 at 5 dots, ~12.3 mm | ~5 mm strip | 1 line at 8–9 pt: just the short ID |
| DK-1209 29×62 (696×271 dots) | v3 at 6 dots, ~14.7 mm (or v3 at 7 dots, ~17 mm) | ~40×23 mm | 4 lines at 9 pt (~28 chars) or 3 lines at 12 pt (~20 chars) |
| DK-1201 29×90 (991×306 dots) | v3 at 7–8 dots, ~17–20 mm | ~60×26 mm | 4–5 lines at 10 pt (~33 chars) |
| 62 mm continuous, cut 62×30 | v3 at 8 dots, ~20 mm | ~35×26 mm, any length | 2 lines at ~20 pt red location code plus 2 lines at 9 pt black detail; make it longer for more |

## 4. Programmatic printing routes from a Linux Docker host

| Route | Works from a Linux Docker host on the LAN? | Notes |
|---|---|---|
| **`brother_ql_next` → raw TCP 9100** | **Yes. Recommended.** | Maintained fork of `brother_ql` (v0.12.0, May 2026, Python ≥ 3.9). QL-820NWB is marked "✓ verified". Network backend: `tcp://192.168.1.21:9100`. Supports `62red`, `--600dpi`, compression and cut control. **Limitation:** "The network backend doesn't support reading back the printer state", so wrong-label or end-of-roll errors aren't detected. [V, README] Brother's own raster reference says raster printing exists for "printing from an operating system other than Windows (Example: … Linux)" and documents a "Standard TCP/IP port" network flow. [V] It needs only an outbound TCP connection from the container to the printer, with no USB passthrough. [I] |
| `brother_ql` (pklaus original) | Works, but unmaintained | Last PyPI release 0.9.4 (Jan 2019). The brother_ql_next README calls it "Original unmaintained version". [V] Don't install both: the README says they're incompatible. [V] |
| `brother_ql-inventree` (matmair fork) | Yes (used by the InvenTree plugin) | "IN MAINTENANCE MODE", with QL-820NWB verified. [V] |
| **`brother_ql_web`** (FriedrichFroebel fork) | Yes, as a print microservice | A web UI and HTTP API: `/api/print/image` accepts an uploaded image. It has a Docker Compose setup and is actively maintained (pushed Oct 2026). [V, README] It's a ready-made container you can point Homebox, Snipe-IT or scripts at. [I] |
| P-touch Template mode (send template number + data over TCP) | Yes, in principle | The printer stores up to 255 templates designed in P-touch Editor (Windows/macOS) and fills them with data sent over the network. [V for the existence of the mode and the 255-template figure, per Brother's P-touch Template Command Reference listing; the full command reference wasn't read] You'd need a desktop OS to design the templates, and the QR rendering happens in the printer. Not recommended: it's harder to version and test than generating PNGs. [I] |
| Brother Linux LPR + CUPS wrapper driver | Possible but awkward | Brother ships deb/rpm and Raspbian builds of the LPR and cupswrapper drivers (v3.1.5-0), and "Brother drivers for Linux distributions support black and white printing only", so **no red**. [V, Brother downloads page] You'd run CUPS inside a container with i386/amd64 binary blobs. No benefit over raw raster. [I] |
| AirPrint / IPP driverless | Unverified | Brother marketing lists AirPrint. [V] Whether CUPS driverless IPP gives correct DK sizes and cutting on Linux wasn't tested. [I] |
| **b-PAC SDK** | **No** | Windows-only COM automation SDK (QL-820NWB supported in v3.2–3.4). [V, Brother b-PAC environment page] |
| Brother Mobile SDK | No (iOS/Android only) | [V, Brother Mobile SDK model list] |

Operational gotchas:

- **Give the printer a DHCP reservation or static IP.** Every route above addresses it by IP. [I]
- **Editor Lite mode** only matters for USB printing ("you need to disable it if you want to print via USB"). [V, brother_ql README] It doesn't affect network printing. [I]
- The printer must be switched to raster mode before raster data is sent ("Be sure to switch to this mode"). brother_ql does this for you. [V, Raster ref p.29]

## 5. Native and plugin support in the candidate tools

| Tool (latest release) | How labels are generated | Prints to the QL-820NWB | What the QR encodes | USB-scanner path |
|---|---|---|---|---|
| **InvenTree** (1.6.1, Oct 2026) | HTML/CSS label templates rendered by WeasyPrint. Size is set with `@page` in mm; `{% qrcode … %}` tag. Plugins: PDF download, label sheet, **Label Machine**. [V, docs] | **Native, via the official `inventree-brother-plugin`** (2.0.6, May 2026; requires InvenTree ≥ 0.18). Set it up under Admin › Machines with model QL-820NWB, label media (including `62red`), IP address, auto-cut, rotation, HQ and compression. It sends over `tcp://<ip>` with the `network` backend. [V, README + source] Die-cut images are padded to the media's printable dots; continuous images are passed as-is. Red is enabled only when media = `62red`. [V, source] | By default an internal barcode in **short format** (`INV-PA10`, `INV-SI123`) or **JSON** (`{"part": 10}`). Built-in barcode plugin only. [V, docs + source] **A phone camera won't open a page from these.** A template can encode a URL in the QR instead, but the built-in scanner only matches the short or JSON formats, so it won't resolve that URL. [V for the plugin parsing; I for the conclusion] | Strong: barcode scan dialogs throughout the UI and the mobile app. [V that scanning exists, per docs; app behavior not checked] |
| **Snipe-IT** (8.8.3, Oct 2026) | New label engine (TCPDF → PDF) with templates for Brother TZe tapes, DYMO, Zebra, Avery sheets and generic tapes. Includes `Tapes\Brother\TZe_62mm_Landscape(_A)`, which suits 62 mm DK. [V, source] | **No direct printing.** Snipe-IT's own QL-820NWB + DK-2251 example: "Generate label", then the browser print dialog with a custom 62×38 mm page, 0 margins and landscape. [V, Snipe-IT "Asset Label Printer Examples"] The client needs a Brother driver. Red would need a red-capable driver path. [I] | QR → "/hardware/{id}", which opens the asset page on a phone. A 1D barcode on the same label is "for the top search bar with a USB scanner". [V, docs] | Search-bar based. Asset-centric; consumables and loanables are weaker. [I] |
| **Homebox** (0.27.1, Oct 2026) | Built-in label maker renders a PNG with a QR code and text. Size is set in pixels (`HBOX_LABEL_MAKER_WIDTH/HEIGHT`, defaults 526×200). Optional **external label service** (`HBOX_LABEL_MAKER_LABEL_SERVICE_URL`) receives the size, text and URL and returns an image. [V, docs + source] | **Indirect.** `HBOX_LABEL_MAKER_PRINT_COMMAND` runs a command with `{{.FileName}}` replaced by the PNG path, run *inside the Homebox container*. [V, docs + source] The stock images are Alpine or distroless with no Python, so you'd need a custom image that bundles `brother_ql_next`, or a command that posts the PNG to a `brother_ql_web` container. [V for the base images; I for the workaround] | **URL** to the item or location page. [V, docs] Note: the QR is generated with its border disabled, so the quiet zone comes from the margin and padding settings. [V, source] | Built-in camera QR reader (v0.18+). USB-scanner batch workflow not documented. [V / not verified] |
| **Part-DB** (2.20.3, Oct 2026) | Label profiles with a WYSIWYG or Twig HTML template rendered to PDF (dompdf). Paper size in mm. Barcodes: QR, DataMatrix, Code39, Code128. [V, docs + source] | **No direct printing.** You print the PDF through the browser and OS. Nothing Brother-specific in the docs or source. [V] | **URL** `https://<host>/scan/<type>/<id>`, which opens the page on a phone. [V, source] | **Best of the four for USB scanners:** keyboard-wedge support, and a scanner prefix of `<SOH>` (0x01) triggers a global redirect from anywhere. The scanner page **parses its own QR URLs** (`^https?://.*/scan/(\w+)/(\d+)/?$`) and also distributor barcodes (Digi-Key, Mouser, LCSC, TME, EIGP114). [V, docs + source] |

**Summary:** InvenTree has the only first-party direct-to-printer path, but its native QR payload isn't a phone-openable URL. Part-DB is the only tool whose QR is both a phone-openable URL and resolvable by its own USB-scanner flow, but it has no direct printing. If the chosen tool lacks direct printing, a small sidecar container fills the gap: either `brother_ql_web`, or a script that fetches the tool's label PDF or PNG through its API and sends it with `brother_ql_next`. [I]

## 6. Open questions for the map [I]

1. **QR payload: URL vs. tool-internal ID.** Phones need a URL. Batch USB scans for restock need something the tool, or our restock processing, can resolve. InvenTree's native codes are IDs, not URLs. Part-DB does both. Decide with the tool choice.
2. **Canonical hostname for labels.** Printed URLs outlive server moves, so use a stable DNS name, not an IP. Also, phones must reach the LAN-only server: are they on lab Wi-Fi or VPN? Otherwise a scanned QR opens nothing.
3. **Print-and-scan test** with the lab's phones on DK-1204 and DK-1209 at 4, 5 and 6 dots/module before buying rolls in bulk.
4. **Silent failures:** the brother_ql network backend can't read printer status, so empty-roll or wrong-roll errors aren't reported. Is that acceptable, or should the printer be USB-attached to the Docker host (pyusb or `linux_kernel` backends can read status)? [V for the limitation]
5. **Roll changes:** the printer holds one roll. Which single stock covers most labels, and who swaps rolls?

## Sources

Brother (primary):

- QL-820NWB specifications: https://support.brother.com/g/b/spec.aspx?c=us&lang=en&prod=lpql820nwbeus
- QL-820NWB consumables (US): https://support.brother.com/g/b/colist.aspx?c=us&lang=en&prod=lpql820nwbeus&cao=roll
- QL-820NWB consumables (UK, EU part numbers): https://support.brother.com/g/b/colist.aspx?c=gb&lang=en&prod=lpql820nwbeuk&cao=roll
- DK-2251: https://www.brother-usa.com/products/dk2251
- DK-2212: https://www.brother-usa.com/products/dk2212
- Software Developer's Manual, Raster Command Reference QL-800/810W/820NWB v1.01: https://download.brother.com/welcome/docp100278/cv_ql800_eng_raster_101.pdf
- P-touch Template Command Reference QL-810W/820NWB (listing): https://support.brother.com/g/s/es/dev/en/command/reference/index.html
- b-PAC operating environment: https://support.brother.com/g/s/es/dev/en/bpac/environment/index.html
- Mobile SDK supported models: https://support.brother.com/g/s/es/dev/en/mobilesdk/model/index.html
- QL-820NWB downloads (Linux LPR/CUPS wrapper): https://support.brother.com/g/b/downloadtop.aspx?c=us&lang=en&prod=lpql820nwbeus

Libraries:

- brother_ql_next: https://github.com/LunarEclipse363/brother_ql_next (`brother_ql/labels.py`, README); PyPI `brother-ql-next` 0.12.0
- brother_ql (original): https://github.com/pklaus/brother_ql; PyPI `brother-ql` 0.9.4
- brother_ql-inventree: https://github.com/matmair/brother_ql-inventree
- brother_ql_web (fork): https://github.com/FriedrichFroebel/brother_ql_web

QR and scanners:

- DENSO WAVE, versions and capacity: https://www.qrcode.com/en/about/version.html and https://www.qrcode.com/en/about/versionPage/versionPage1_10.html
- DENSO WAVE, code area and quiet zone: https://www.qrcode.com/en/howto/code.html
- Zebra DS2200 series spec sheet: https://www.zebra.com/us/en/products/spec-sheets/scanners/general-purpose-scanners/handheld/ds2200-series.html

Inventory tools:

- InvenTree label docs: https://docs.inventree.org/en/latest/report/labels/
- InvenTree internal barcodes: https://docs.inventree.org/en/latest/barcodes/internal/
- InvenTree barcode plugin source: https://github.com/inventree/InvenTree/blob/master/src/backend/InvenTree/plugin/builtin/barcodes/inventree_barcode.py
- inventree-brother-plugin: https://github.com/inventree/inventree-brother-plugin (`inventree_brother/brother_plugin.py`)
- Snipe-IT label printer examples: https://snipe-it.readme.io/docs/-asset-label-printer-examples-and-settings
- Snipe-IT barcodes / asset labels: https://snipe-it.readme.io/docs/barcodes, https://snipe-it.readme.io/docs/asset-labels
- Snipe-IT label templates: https://github.com/grokability/snipe-it/tree/master/app/Models/Labels
- Homebox configuration, tips and external label service: https://github.com/sysadminsmedia/homebox/tree/main/docs/src/content/docs/en (`quick-start/configure/index.mdx`, `user-guide/tips-tricks.mdx`, `advanced/external-label-service.mdx`); source `backend/pkgs/labelmaker/labelmaker.go`
- Part-DB labels and scanner: https://docs.part-db.de/usage/labels.html, https://github.com/Part-DB/Part-DB-server/blob/master/docs/usage/scanner.md; source `src/Services/LabelSystem/`
