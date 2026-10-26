# CPE Lab Inventory

The inventory, checkout and ordering system for the CPE design lab: what the lab owns, where it lives, who has it, and what needs ordering.

## Kinds of stock

**Consumable**:
Stock that students take freely and don't return, e.g. resistors, dupont wires, adhesives. Counting it is optional and only serves restocking.
_Avoid_: Supply, part

**Loanable**:
Stock whose units are interchangeable and tracked by count, which students check out and return, e.g. Raspberry Pis, ESP32s, lidar.
_Avoid_: Asset, checkout item

**Equipment**:
Stock tracked as individually numbered Units, usually installed at a bench and occasionally loaned, e.g. oscilloscopes or bench power supplies.
_Avoid_: Asset, tool, instrument

**Unit**:
One numbered piece of Equipment, e.g. SCOPE-01, with a home bench and a status (at bench, on Loan, out of service).
_Avoid_: Asset, serial

## Catalog

**Catalog Item**:
One kind of thing the lab holds, e.g. "Raspberry Pi 5 8GB" or "10kΩ ¼W resistor". It has exactly one stock kind and exactly one Category.
_Avoid_: Part, product, SKU, component

**Category**:
A node in the lab's function-based tree (what a thing does), as deep as a branch needs. It's independent of stock kind.
_Avoid_: Type, class

**Location**:
A place stock can be, nesting to any depth: room → cabinet → shelf → drawer/bin, plus benches. It has a short, readable ID such as `A-07`.
_Avoid_: Bin (a bin is one kind of Location), container

**Placement**:
A Catalog Item held at one Location, with its own on-hand count. An item can have several Placements, and a Location can hold several items.
_Avoid_: Stock lot, slot

**Tag**:
A cross-cutting label on a Catalog Item, such as platform or compatibility (`raspberry-pi`, `fits:pi5`), for facts the single Category can't hold.
_Avoid_: Label (that word means the printed sticker)

## Activities

**Loan**:
A Lab Staff-approved record of one person holding some count of a Loanable, or a specific unit of Equipment, until a due date. It may be made on a Project's behalf.
_Avoid_: Checkout, rental, requisition

**Project**:
A student project the lab supports, with a name, an end date, an optional Course, and a Team. It's kept in the system, not in an outside sheet.
_Avoid_: Group

**Team**:
The Lab Users who belong to a Project. Any member can return that Project's Loans.

**Cubby**:
A Team's shelf slot where Lab Staff leave the items of a fulfilled Loan for collection.
_Avoid_: Locker, bin

**Write-off**:
A unit leaving lab stock for good, recorded at check-in as damaged or lost. Damaged Equipment is marked out of service instead.
_Avoid_: Disposal, retirement

**Semester**:
A named term with an end date, kept by Lab Staff (e.g. Fall 2026). It sets the default due date for Loans without a Project.
_Avoid_: Term

**Restock Report**:
One person's signal that a Consumable or Loanable is low or out, raised by scanning its label with the in-app scanner (no sign-in needed) or in a Scan Session.
_Avoid_: Alert, low-stock ticket

**Restock Need**:
The single open entry on the restock list for a Catalog Item, grouping its Restock Reports. It moves Open → Ordered → Restocked, or is Dismissed.
_Avoid_: Restock request, restock ticket

**Scan Session**:
A Lab Staff run with the USB handheld scanner, scanning many labels in one mode ("Needs restock" or "Count") and then submitting the reviewed batch.
_Avoid_: Sweep, audit

**Purchase Request**:
A Lab User's request for the lab to buy one or more Line Items, with a single purpose. What the lab buys stays lab-owned: items for a Project arrive as stock already on Loan to the requester.
_Avoid_: Order request, requisition

**Line Item**:
One product on a Purchase Request (quantity, product, unit price), approved or denied on its own.
_Avoid_: Line, entry

**Order List**:
Approved Line Items and open Restock Needs grouped by vendor, ready for Lab Staff to order outside the system.
_Avoid_: Cart, batch

**Preferred Vendor**:
A supplier the lab normally buys from. Purchase Requests from other vendors are allowed but get extra scrutiny.
_Avoid_: Supplier, distributor

## People

**Lab Staff**:
The people who run the lab (the Lab Manager and Lab Assistants), who approve Loans and Purchase Requests.
_Avoid_: Admin, approver

**Lab Manager**:
The member of Lab Staff who owns the lab's inventory, catalog and ordering.

**Lab Assistant**:
A member of Lab Staff who runs day-to-day inventory, Loans and purchasing alongside the Lab Manager.

**Permission**:
A single capability granted to a person, e.g. approving Loans or editing Categories. What someone may do is the set of Permissions they hold, from their Role plus any individual grants.
_Avoid_: Right, access

**Role**:
A named, editable bundle of Permissions (Lab Manager, Lab Assistant, Course Staff, Lab User) given to a person.
_Avoid_: Group, user type

**Course Staff**:
Instructors and TAs assigned to a Course. They manage that Course's Projects and Teams but don't approve anything.
_Avoid_: TA, faculty (as roles)

**Course**:
One offering of a class in one Semester, e.g. "Fall26-Design". A Project that spans two semesters belongs to its first offering.
_Avoid_: Class, section

**Lab User**:
Anyone outside Lab Staff who borrows, raises Restock Reports or submits Purchase Requests. Usually a student in a course, but not always.
_Avoid_: Student (as a role), borrower
