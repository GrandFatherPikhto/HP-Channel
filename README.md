# HP-Channel-v099 — UCC21750 driver for the CM200DU-24NFH IGBT module

*Читать по-русски: [README_ru.md](README_ru.md)*

The board drives one Mitsubishi CM200DU-24NFH half-bridge module from the HiPiMS-Controller-STM32-v099 controller
(Cyclone 10 LP). The controller does not change: the board plugs into its HP connector (IDC 2×7) exactly like the
power board with C2M0080120D SiC transistors. The goal is to keep the installation able to run on CM200DU-24NFH
modules, but with an isolated driver and desaturation protection instead of the non-isolated IR2110 / IR2213
drivers (§8, item 6).

The description follows the schematic as of netlist `HP-Channel-v099.net`, 16.11.2026 12:40. §3, §5 (items 1, 7)
and §6 were checked against the board and the netlist of 21.11.2026 17:40.

* §1–§3 — the module, the interfaces and the connections;
* §4 — design decisions already made;
* §5 — what is not in the schematic yet;
* §6 — layout and isolation: clearances, conformal coating, where 800 V can break through, running without the
  low-side arm;
* §7 — an open question about the controller;
* §8 — **questions about the installation that the component values depend on**.

The UCC21750 parameters were checked against the TI datasheet (2023 revision). Values marked **(check)** are given
from memory and must be checked against the relevant documents before ordering.

---

## 1. The CM200DU-24NFH module

```text
C1 ──┬── IGBT1 (G1/E1) ──┬── C2E1 ──┬── IGBT2 (G2/E2) ──┬── E2
     (DC+)                (output: midpoint)              (DC−)
```

* Power terminals (M6), per the drawing: **C2E1 — E2 — C1**.
* Control terminals — two pairs of 2.8 mm Faston tabs: **G1 + E1** (high-side IGBT), **G2 + E2** (low-side). E1
  and E2 are auxiliary (Kelvin) emitters; inside the module they are the same nodes as C2E1 and E2.

Datasheet parameters that shape the driver:

| Parameter | Value | What it means for the driver |
| --- | --- | --- |
| VCES | 1200 V | The DESAT circuit and the isolation clearances must take the full bus voltage |
| VGES | ±20 V | A +15 / −5 V gate supply is within limits |
| IC / ICM | 200 A / 400 A (pulse) | |
| VCE(sat) | 5.0 V typ., 6.5 V max. at 200 A, VGE = 15 V | Taken into account in the DESAT threshold (§4) |
| QG | 900 nC | Peak gate currents are a few amperes (§5) |
| External RG | 1.6 … 16 Ω | The chosen 10 Ω / 3.3 Ω are within the range |
| td(on) / tr | ≤ 300 ns / ≤ 80 ns | Fast entry into saturation — a short DESAT blanking time is acceptable |
| Note 4 | No short circuit capability is designed | Short-circuit withstand is not rated — the desaturation protection must trip as fast as possible |
| Switching test conditions | VGE = ±15 V | At −5 V the margin against false turn-on is smaller (§5, item 6) |

---

## 2. Interface to the controller (J1, IDC 2×7)

The board connects to the controller **through the HP-ISO-Channel adapter board** (`~/Projects/KiCad/HiPIMS/HP-ISO-Channel-v099`):

```text
controller J10/J11 (IDC 2×6, 3.3 V) ─► HP-ISO-Channel: ISO7740 / ISO7720 + B0505S ─► J2 (IDC 2×7) ─► J1 of this board
                            isolated domain +5V_IDC / GND_IDC
```

* HP-ISO-Channel galvanically separates the controller from the driver's primary side: arc currents and
  common-mode noise of the power stage do not reach the FPGA ground. The UCC21750 separates the primary side from
  the high-voltage arms.
* The `+5V_IDC` / `GND_IDC` domain is powered by the isolated B0505S DC/DC on HP-ISO-Channel; the CTL / RST signals
  come from SN74LVC1G17 buffers powered from +5V_IDC, and FLT is received by the same kind of buffer from +5V_IDC
  (with a 100 kΩ pull-down to GND_IDC on HP-ISO-Channel: a disconnected cable reads as a fault).
* So the logic on this connector is **5 V**, and the UCC21750 primary side is powered from +5V_IDC.

The J1 pinout matches J2 of HP-ISO-Channel. Signal lines alternate with ground.

| Pin | Net | Purpose |
| --- | --- | --- |
| 1, 3, 5, 7, 9, 11, 13 | `GND_IDC` | Isolated ground of the HP-ISO-Channel domain |
| 2 | `+5V_IDC` | UCC21750 primary-side supply (from the B0505S on HP-ISO-Channel) |
| 4 | `/CTL_A` | Channel A control (high-side IGBT) |
| 6 | `/RST_A` | Channel A fault reset / enable |
| 8 | `/FLT_A` | Channel A fault: open drain, active low |
| 10 | `/CTL_B` | Channel B control (low-side IGBT) |
| 12 | `/RST_B` | Channel B reset / enable |
| 14 | `/FLT_B` | Channel B fault |

**A direct connection to the controller's HP connector (J10 / J11) is not supported.** The connector fits
mechanically, but the controller's logic is 3.3 V (74HC14 powered from +3.3 V): the 5 V FLT pull-up on this board
exceeds the controller's input voltage limit, and the datasheet does not guarantee the UCC21750 input thresholds at
VCC = 5 V for a 3.3 V signal.

---

## 3. Connection to the module

Channel **A is the high-side IGBT (IGBT1)**, channel **B is the low-side one (IGBT2)**.

The installation's bus voltage is **800 V**.

Each arm has **one 4-position JST VH connector (B4P-VH)**: J2 for arm A, J3 for arm B. Position 2, between the
collector and the emitter, is empty: there is no pad 2 on the board, and pin 2 is removed from the header. Crimp
contacts were chosen over screw terminals because a screw clamp loosens over time and needs re-tightening.

| Connector | Pin | Board net | Module terminal |
| --- | --- | --- | --- |
| J2 (A) | 1 | `/SwitchA/DRAIN` (collector, DESAT) | C1 (M6 power terminal, DC+) |
| J2 (A) | 2 | — (no pad, pin removed) | — |
| J2 (A) | 3 | `/SwitchA/SOURCE` (emitter) | E1 (auxiliary) |
| J2 (A) | 4 | `/SwitchA/CLMPI` (gate) | G1 |
| J3 (B) | 1 | `/SwitchB/DRAIN` (collector, DESAT) | C2E1 (M6 power terminal, midpoint) |
| J3 (B) | 2 | — (no pad, pin removed) | — |
| J3 (B) | 3 | `/SwitchB/SOURCE` (emitter) | E2 (auxiliary) |
| J3 (B) | 4 | `/SwitchB/CLMPI` (gate) | G2 |

The DRAIN / SOURCE net names are kept from the SiC version; for an IGBT they are the collector / emitter.

* **C–E clearance at the connector.** Across C and E of an arm that is off sits the whole bus (800 V) plus the
  turn-off overshoot. This is functional insulation inside one circuit; the connector rating (250 V between
  adjacent contacts) does not apply to it — the distance decides. Contact centres 1 and 3 are 7.92 mm apart, the
  pins ≈ 6.8 mm apart in air, the pads 5.22 mm apart on the board. Per IPC-2221B that is ≈ 1040 V uncoated and more
  under conformal coating (§6.2) — with margin.
* **Pad 2 is missing on purpose.** A floating pad in the middle would cut the collector → emitter creepage path
  from 5.2 mm to 2 × 1.26 ≈ 2.5 mm.
* **Connect only with the bus discharged.** Contact 1 is at collector potential.
* **One connector per arm, not per module.** The high-side and low-side gates sit at different potentials (the
  midpoint and DC−). A shared header would join the A and B isolation zones on the board; so A and B are separate
  connectors, each in its own zone.
* **Position 2 of the mating housing carries no contact**, so the clearance is not defeated.
* **The collector wire must be high-voltage** (silicone or PTFE, 2–3 kV). Check that its cross-section and the
  outer insulation diameter fit the VH crimp contact (SVH) **(check)** against the contact's datasheet.
* **Gate / emitter** — a twisted pair, as short as possible. On the module side: G and E — insulated 2.8 mm
  Fastons; C — a ring terminal for the M6 power screw.
* **A / B marking.** The connectors are identical: on the silkscreen and on the cables — "A → C1 / E1 / G1",
  "B → C2E1 / E2 / G2". Swapped cables swap the arms' functions, and DESAT would measure the other arm.
* **The 24 V input (J5) is a DC barrel jack** instead of a 2-pin JST VH. The arm connectors are VH too, so a
  2-pin power plug could be pushed into a gate connector off by one position; a barrel jack makes that physically
  impossible. See §5, item 7.
* **CLMPI** (the active Miller clamp) is connected on the board to the gate node after R3 / R4 (R13 / R14). It is
  effective with short wires to the module (a few centimetres). With long wires the main protection against false
  turn-on is the negative gate bias.
* **The high-side arm floats:** the secondary side of channel A, including its DC/DC, sits at the midpoint
  potential and swings by the full bus voltage at every switching edge.
* **Without the low-side arm** the module's G2 gate must not be left floating — see §6.4.

---

## 4. Design decisions already made

* **Hardware half-bridge interlock.** The IN− input of channel A is connected to `/CTL_B`, the IN− of channel B
  to `/CTL_A`. The high-side and low-side IGBTs cannot be on at the same time, whatever the firmware does.
* **Fault signal.** The RDY and FLT outputs of each UCC21750 (open drain) are tied together: a secondary-side
  supply dropout is also reported to the controller as a fault.
* **Desaturation protection (DESAT):** two US1M diodes in series (2 kV), a 470 Ω resistor (1210), a 47 pF
  blanking capacitor, a BAT54S clamp between the emitter and +15 V.
  * UCC21750 parameters (TI datasheet, 2023 revision): charge current ICHG = 430 / 500 / 570 µA, threshold
    VDESAT = 8.5 / 9.15 / 9.8 V, internal leading-edge blanking tDESATLEB = 200 ns.
  * Blanking time ≈ 47 pF · 9.15 V / 500 µA ≈ 0.86 µs (0.7…1.07 µs over the spread) plus 200 ns. Given the
    module's fast turn-on (td(on) + tr < 0.4 µs) and the lack of a rated short-circuit withstand, it should not
    be made longer.
  * Threshold: the voltage at the DESAT input ≈ VCE + 2·Vf + 0.5 mA·470 Ω ≈ VCE + 1.8 V. With VDESAT = 9.15 V it
    trips at about VCE ≈ 7.3 V (6.7…8.0 V over the threshold spread), which on the output characteristic
    (VGE = 15 V) corresponds to ≈ 300 A. Whether that level is acceptable depends on the peak discharge currents (§8).
* **Gate resistors:** 10 Ω for turn-on (OUTH), 3.3 Ω for turn-off (OUTL).
* **Power:**
  * the secondary sides — isolated Murata MGJ2 +15 / −5 V DC/DCs (U1, U2) from a separate 24 V input (J5);
  * the 24 V domain is isolated both from the controller and from the arms;
  * the UCC21750 primary side is powered from the +5 V of the controller connector.
* AIN is tied to COM, APWM is unused. Decoupling capacitors sit at both sides of each UCC21750 and at each DC/DC.

---

## 5. What is not in the schematic yet

1. ~~Connectors to the module~~ — **done**: J2 / J3 B4P-VH, 1 — collector (`DRAIN`), 2 — none (pad and pin
   removed), 3 — emitter (`SOURCE`), 4 — gate (`CLMPI`), as in §3. (In the netlist of 16.11 14:18 they were J3 / J4
   with a different pinout; renamed and re-pinned by 21.11.)
2. **Done (netlist 16.11 14:45): R2 / R12.** **A 10 kΩ gate–emitter resistor on each arm** (between
   `/SwitchX/CLMPI` and `/SwitchX/SOURCE`) — holds the IGBT off with the connector unplugged or the driver
   unpowered. A resistor directly across the module's G–E terminals is also desirable.
3. **Logic levels are 5 V, matched to HP-ISO-Channel (§2); no LDO is needed.** The supply still has to be checked:
   * the B0505S is an unregulated converter: at light load its output rises. For the UCC21750 VCC the recommended
     range is 3.0…5.5 V, the absolute maximum 6 V; FLT and RDY must not exceed VCC. **Measure `+5V_IDC`** on an
     assembled HP-ISO-Channel with this board connected (the domain load is about 20–30 mA: two UCC21750, the
     buffers, ISO77xx, an LED, the 1 kΩ load resistor R3). Above 5.5 V — add pre-load or put a 5 V regulator after
     the B0505S.
4. **Input pull-ups/downs.** The UCC21750 has internal ones: IN+ to GND ≈ 55 kΩ, IN− to VCC ≈ 55 kΩ, RST/EN to
   GND ≈ 50 kΩ; with a broken cable the output is held low and the driver is off. External 10 kΩ to `GND_IDC` on
   IN+ and RST/EN are not required; with a long cable to the controller they help as a stiffer pull-down against
   pick-up.
5. **Done (14:45): R3 / R4, R13 / R14 in 1206.** **Gate resistors in a 1206 package, preferably pulse-rated.**
   Peak turn-off current ≈ (15 + 5) V / 3.3 Ω ≈ 6 A at QG = 900 nC.
6. **Negative gate bias.** The module's switching characteristics are rated at ±15 V. If an MGJ2-series converter
   with +15 / −8 (−9) V in the same package is available, it is preferable to −5 V.
7. **The 24 V input is a DC barrel jack** instead of a 2-pin JST VH. Done (14:45): J5 GCT DCJ200-10-A
   (5.5 / 2.1 mm).
   * a jack rated at least 24 V, with current margin for two MGJ2 **(check)**;
   * the centre contact is "+": the ⊖-◎-⊕ polarity symbol on the silkscreen next to J5;
   * no reverse-polarity protection is fitted: the board is powered from the installation's own supply. If any
     adapter may be used (some have "−" in the centre), a series Schottky diode is needed before C1 — the MGJ2
     inputs are not protected against reverse polarity.

---

## 6. Layout and isolation

The board has two layers. The isolation clearances are set by the rules file `HP-Channel-v099.kicad_dru` (the
domains are defined by net names, no net classes needed). On refill (B) the pours pull back by these rules on
their own; DRC checks them.

### 6.1 Domains and clearances

* **Primary side:** `GND_IDC`, `+5V_IDC`, `/CTL_x`, `/RST_x`, `/FLT_x` and the UCC21750 inputs that live on the
  arm sheets (`/SwitchX/CTL_P`, `/SwitchX/CTL_N`, `/SwitchX/RST_SN`); separately — the 24 V domain (`+24V`,
  `GND_24V`).
* **Arm A:** `/SwitchA/*` and the nets of its LEDs and diodes; **arm B:** `/SwitchB/*` and the same.
* **The collector net** — everything at collector potential: `/SwitchX/DRAIN` and the DESAT chain beyond the
  first diode's cathode — `Net-(D4-K)`, `Net-(D5-K)` (arm A), `Net-(D10-A)`, `Net-(D10-K)` (arm B). On the board this
  is one small spot per arm: contact 1 of the connector, R11 / R21, D4–D5 / D9–D10.

| Between | Clearance | Rule in `.kicad_dru` |
| --- | --- | --- |
| arm A ↔ arm B | 2 mm | `arm A to arm B` |
| arm ↔ primary side | 2.5 mm | `arm A to primary`, `arm B to primary` |
| collector net ↔ the rest of its arm | 2.5 mm | `HV to its arm` |
| IDC ↔ 24 V | 0.5 mm (functional insulation) | `IDC to 24V` |
| a DESAT diode's own pads (SMA) | 1.4 mm | `DESAT diode own pads` |

Between the gate (`CLMPI`) and the emitter (`SOURCE`) of one arm there are at most 20 V (+15 / −5 V): the ordinary
clearance applies there.

### 6.2 Why the clearances are what they are, and why coating is required

* IPC-2221B, table 6-1, external conductors above 500 V: **uncoated (B2) — 0.005 mm/V**, **under a permanent
  polymer coating (B4) — 0.00305 mm/V**.
* 2.5 mm under coating is ≈ 820 V: at an 800 V bus the margin is only 2–3 %. The same 2.5 mm uncoated is 500 V.
* 2 mm under coating is ≈ 650 V: the clearance between the arms is about 20 % below IPC, accepted deliberately
  (see below). It only matters when J3 is connected (§6.3, item 3).
* So **the whole board is conformally coated**; without coating these clearances are not acceptable. The figures
  refer to the working voltage; the table does not cover short turn-off overshoots.
* The basis for the small clearances is experience: a driver on the IR2213 (2.54 mm between pins) ran in the
  installation for about 10 years without breakdowns.
* The closely spaced UCC21750 pins are no argument here: adjacent pins on one side (1.27 mm pitch) are nets of one
  arm with at most 20 V between them; across the barrier the DW-16 package gives 7.5 mm between pins; the DESAT pin
  is low-voltage — 800 V only appears beyond the cathode of D4 (D9), outside the chip.

### 6.3 Where 800 V can break through

While the high-side switch is off:

1. **Inside arm A: collector net → the rest of the arm.** The main direction. DRAIN, the cathodes of D4 / D5 and
   R11 sit at +800 V relative to SOURCE, and 2.5 mm away is everything referenced to the emitter:
   * **into the gate (`CLMPI`)** — the IGBT gate oxide breaks down, the switch turns on or fails short, the UCC21750
     outputs burn;
   * **into `SOURCE`** — the bus current bypasses the IGBT through a board trace; the desaturation protection does
     not see it (no current flows through the switch); the trace vaporises, a fire is possible;
   * **into +15 / −5 V** — the MGJ2 output and the UCC21750 supply burn.

   The same applies to arm B when it is fitted and the low-side switch is off.
2. **Arm A → primary side.** The whole secondary side of A floats with the midpoint between DC− and DC+. If the
   controller or DC− is earthed, up to 800 V can appear between arm A and `GND_IDC`. The weak spot is where the
   pours come within 2.5 mm; the UCC21750 and the MGJ2 have their own barriers. HP-ISO-Channel stands further along,
   so this will probably not reach the FPGA, but it will destroy the primary side of this board.
3. **Arm A → arm B (2 mm).** Depends on J3. If it is connected to G2 / E2, arm B's copper sits at DC− and arm A at
   the midpoint: up to 800 V across 2 mm, below IPC even under coating. If J3 is not connected, arm B's copper
   floats and there is no hazard.

### 6.4 Running without the low-side arm

The CM200DU-24NFH is a half bridge: the low-side IGBT stays in the package even if arm B is not used. **The G2 gate
must not be left floating:** the dv/dt edge of every high-side switching event charges it through the Miller
capacitance, the low-side switch partly turns on, and shoot-through current flows through the half bridge.
Options:

* **short G2 to E2 right at the module** — with a jumper across the Fastons or a resistor of 10 kΩ or less. Then do
  not connect J3: no cable runs to arm B, and item 3 of §6.3 disappears. R12 on the board does not help here — it
  sits on the board, not on the module;
* **or fit arm B completely and hold `/CTL_B` low:** the driver keeps −5 V on the gate and the Miller clamp works.
  More robust, but needs the second MGJ2 and all of its parts.

### 6.5 Other notes

* **There are no slots** under the UCC21750 or at the connectors: isolation relies on distance and coating.
* **Mounting holes:** H1 / H2 are on `GND_IDC`, H3 / H4 have no net.
* **The gate loop** (OUTH / OUTL → R3 / R4 → gate → module → emitter → COM / VEE) — minimum area, 0.65 mm
  traces.
* **Open items:**
  * the coating area on a user layer, labelled "Conformal coating";
  * the 100 nF VDD / VEE decoupling (C9 / C10, C25 / C26) is on the bottom, 4.2 mm from pins 5 and 8, through
    vias; better on top, within 1–2 mm;
  * C19 / C35 (DESAT blanking, 47 pF) are 6–8 mm from the DESAT pin; move them next to the DESAT / COM pins,
    otherwise pick-up on the long branch may cause false protection trips.

---

## 7. Controller: output state at power-up

According to the available information **(check)** in the Cyclone 10 LP Handbook, the FPGA user pins have weak
pull-ups during configuration. There are currently no resistors between the FPGA and the 74HC14 buffers on the
`HP.x.CTL` / `HP.x.RST` lines. In that case the CTL signals of both channels are active while the FPGA loads.
The interlock on the driver board (§4) rules out shoot-through in the half bridge, but one of the switches can still
turn on at power-up — for this board and for the SiC version alike.

The proposed fix is ~2.2 kΩ resistors to ground on all `HP.x.CTL` and `HP.x.RST` lines on the FPGA side
(8 resistors).

---

## 8. Questions about the installation

The DESAT values, the DC/DC choice and the mechanical design depend on the answers:

1. **Peak discharge pulse current** (and the arc current before the protection trips): does it stay within the
   ≈ 300 A the DESAT threshold is set for (§4)?
2. **Bus voltage** — 800 V (taken into account in §3 and §6). Find out the size of the turn-off and arc overshoots,
   if they were measured: at 800 V the clearances of §6 work with almost no margin.
3. **Pulse length and repetition rate** — to estimate the driver power and the module heating.
4. **Which switches of the module are used** in the installation: both (half bridge) or one; what the second one
   does. If the low-side arm is not fitted — how the G2 gate is shorted (§6.4).
5. **Distance from the driver board to the module:** can the board be mounted directly on the module terminals
   (this decides how useful the Miller clamp is and what negative bias is needed)?
6. **Experience with the previous versions:** first an IR2110 was fitted — it failed systematically; then the
   IR2213, which ran for about 10 years. In which modes did the IR2110 fail (arcs, power-up, long operation), and
   what changed with the IR2213? The design assumes the main cause was the lack of isolation and desaturation
   protection; confirmation or a different picture will affect the decisions of §4–§5.
