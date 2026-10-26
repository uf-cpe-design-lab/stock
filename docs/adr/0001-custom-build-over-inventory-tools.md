# Build a custom system instead of adopting an inventory tool

We build our own lab inventory app rather than adopting InvenTree, Shelf.nu, Snipe-IT, Part-DB or Homebox. None of them handles approved Loans by count with reservations and Project due dates, Restock Reports without sign-in, Scan Sessions, multi-line Purchase Requests that arrive already on Loan, or Course-scoped Permissions. We'd have to build all of that on any base. What a tool would add (catalog, locations, counts, label printing) is the easy part at under 500 kinds of stock. The Lab Manager builds and maintains it with AI help.

## Considered Options

- **InvenTree + lab plugin:** the strongest platform, with an official QL-820NWB plugin. But its maintainers ruled Loans out of scope (inventree/InvenTree#9068), and its group permissions are too coarse, so the core workflow would be plugin code tied to InvenTree's upgrades.
- **Shelf.nu + side app:** the best fit for Loans, but it has no public API to extend, and self-hosting means running Supabase under a community Docker image.
- **Snipe-IT + companion app:** labels and scanning work only for individually tagged items (none for bins), and count-based loans have no due dates.

Full comparison: research branch `research/inventory-tool-fit`, `docs/research/inventory-tool-fit.md`.
