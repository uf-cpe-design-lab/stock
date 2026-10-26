# PROTOTYPE: lab category taxonomy draft (throwaway)

Answers the "How deep should the category taxonomy be?" ticket. This is the revised draft after review.

**Rules settled:**
- A custom tree organized by **function** (what the thing does), with no link to Digi-Key's taxonomy.
- **Variable depth:** go as deep as a branch needs. Two levels is the usual start.
- Each Catalog Item has **exactly one** Category. **Tags** cover the cross-cutting facts: platform, compatibility, e.g. `raspberry-pi`, `fits:pi5`.
- Category is **independent** of stock kind (Consumable / Loanable / Equipment).
- Editing Categories is gated by a Permission.

Typical stock kind in brackets: C = Consumable, L = Loanable, E = Equipment.

## Passives [C]
- Resistors > Through-hole, SMD
- Potentiometers & trimmers
- Capacitors > Ceramic, Electrolytic, Tantalum
- Inductors
- Crystals & oscillators
- Assortment kits

## Semiconductors [C]
- Diodes & rectifiers
- Transistors & MOSFETs
- LEDs
- Voltage regulators & power ICs
- Logic & interface ICs
- Op-amps & analog ICs
- Microcontroller chips (bare)

## Compute [L]
- Single-Board Computers: Raspberry Pi, Jetson, BeagleBone… (platform is a tag)
- Microcontroller Boards: ESP32, Arduino, RP2040, STM32… (family is a tag)
- AI Accelerators, e.g. Raspberry Pi AI HAT+ (tags: `raspberry-pi`, `fits:pi5`)
- Expansion boards: HATs, shields, capes (tags give the platform)
- Programmers & debuggers
- Cases, PSUs & heatsinks

## Sensors & Modules [L/C]
- Distance & lidar
- Cameras
- IMUs & motion
- Environmental
- Wireless modules
- Breakouts, misc.

## Displays [L/C]
- Touchscreen displays
- LCD / OLED modules
- E-paper
- Monitors

## Motors & Actuators [L/C]
- DC motors, Servos, Steppers
- Motor drivers

## Power & Batteries [C/L]
- Batteries > LiPo / Li-ion, Alkaline / coin
- Battery holders & connectors
- Wall adapters & USB power
- Chargers

## Prototyping [C]
- Breadboards
- Jumper / dupont wires
- Perfboards & protoboards
- Headers & pin strips
- Terminal blocks

## Wire & Cables [C/L]
- Hook-up wire > Solid core, Stranded
- Ribbon cable
- USB, Video and Network cables
- Adapters & dongles

## Connectors [C]
- Crimp housings (JST, Molex…)
- Barrel jacks
- Crimp terminals & ferrules

## Hardware & Materials [C]
- Standoffs, screws, nuts, washers
- Heat shrink
- Zip ties & cable management
- Tapes
- Adhesives & glues
- Thermal paste & pads

## Tools [L]
- Hand tools: cutters, pliers, strippers, screwdrivers
- Crimpers
- Solder, flux & wick [C]
- ESD supplies [C]

## Bench Equipment [E]
- Oscilloscopes
- Multimeters
- Bench power supplies
- Function generators
- Logic analyzers
- Electronic loads
- Soldering stations
