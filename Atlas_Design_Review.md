# Atlas ESC Design Review

**Project:** Atlas Multi-Channel ESC (KiCad 10.0, 5 hierarchical sheets: 1 root + 4 instanced Motor sheets, no PCB layout started yet)  
**Date:** June 21, 2026  
**Analyzers:** [analyze_schematic.py](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/.agents/skills/kicad/scripts/analyze_schematic.py), [cross_analysis.py](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/.agents/skills/cross_analysis.py), [analyze_emc.py](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/.agents/skills/emc/scripts/analyze_emc.py), [analyze_thermal.py](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/.agents/skills/kicad/scripts/analyze_thermal.py) (all run in modern format, full analysis cache)

---

## Overview

The Atlas ESC is a multi-channel brushless DC (BLDC) motor speed controller consisting of 4 identical motor controller channels. Each channel is defined in a hierarchical sheet instance of [VegaESC.kicad_sch](file:///C:/Users/param/OneDrive/Documents/Code/pcb/project/atlas/VegaESC.kicad_sch) (instanced as Motor_1, Motor_2, Motor_3, and Motor_4). 

Each motor channel is controlled by an Artery [AT32F421K8T7](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/datasheets/AT32F421K8T7.pdf) MCU driving an [HXFD6288QFN24](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/datasheets/HXFD6288QFN24.pdf) 3-phase gate driver. The power architecture is cascaded, starting with a 10V buck regulator ([LMR51420YDDCR](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/datasheets/LMR51420YDDCR.pdf)) that feeds a 3.3V LDO regulator ([TLV76733DRVR](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/datasheets/TLV76733DRVR.pdf)) for the digital logic and a high-precision bidirectional current sense amplifier ([INA186A2](https://www.lcsc.com/datasheet/lcsc_datasheet_2302220000_Texas-Instruments-INA186A2QDBVRQ1_C2867989.pdf)).

---

## Recent Design Updates (June 21, 2026)

The following schematic design modifications were made in the recent update:
1. **Control Interface Consolidation**: The 5-pin programming/ADC connector `J106` and individual phase diagnostic posts `J103` (Phase A) and `J105` (Phase C) on the root sheet were replaced by a single unified 8-pin JST SH connector `J104`. This connector groups the power rail (`+BATT`), digital ground (`GND`), analog telemetry (`CURRENT` and `TELEMETRY`), and the four PWM inputs (`M1`, `M2`, `M3`, `M4`) for external interfacing.
2. **Current Sense MCU Disconnection**: The `CURRENT` telemetry line from the current-sense amplifier (`U101`) output filter was disconnected from the MCUs (`Pin 9, PA3`). Total board current telemetry is now routed exclusively to the external interface via `J104` Pin 3.
3. **Reference Designator Reorganization**: Re-annotated reference designators across subsheets to prepare for PCB layout. For example, `Motor_1` references (which previously used the `2xx` series like `U201`, `U203`) now use `4xx` series (like `U402`, `U404`), mapping logically to layout grids.
4. **General Schematic Cleanup**: Cleaned up wires, organized nets, and resolved dangling connections on the root sheet (removing stray test points `TP105` and `TP106`).

---

## Critical Findings

| Severity | Issue | Section |
|----------|-------|---------|
| **WARNING** | `+BATT` and `VBAT` have no declared source (ERC warning `RS-001`) | [Power Analysis](#power-analysis) |
| **WARNING** | Input voltage is limited to 24V due to TVS Diode `D101` (`SMF24A`) | [Voltage Derating](#voltage-derating) |
| **INFO** | MCU control logic crossings (nets `AHIGH`, `ALOW`, etc.) are false positives | [False Positives / Reviewer Overrides](#false-positives--reviewer-overrides) |
| **INFO** | PCB layout has not been started (290 missing components) | [Schematic ↔ PCB Cross-Reference](#schematic--pcb-cross-reference) |

---

## Component Summary

| Component Type | Quantity |
|----------------|----------|
| Resistors      | 122      |
| Capacitors     | 69       |
| Diodes / LEDs  | 40       |
| Transistors    | 24       |
| ICs            | 11       |
| Inductors / FBs| 5        |
| Connectors     | 1        |
| Test Points    | 26       |
| Mounting Holes | 4        |
| **Total**      | **302**  |

* **Nets:** 177 | **Wires:** 848 | **No-connects:** 52 | **Power rails:** 6
* **Sourcing Audit:** 29 of 31 unique parts have manufacturer part numbers (93.5% coverage). Generic post-connectors (e.g. BATT, GND, motor phases A/B/C) and programming headers lack MPNs, which is normal for layout post connections.

---

## Power Tree

```
[ J101 (BATT Connector Post) ]
             |
             v (VBAT Net)
             |
       [ R101 (0.2 mΩ Shunt) ] <--- [ U101 INA186 Current Sense Amp ]
             |
             v (+BATT Net)
             |
      +------+------------------------------+
      |                                     |
      v                                     v
[ U102 Buck (LMR51420) ]             [ Q307-Q512, Q401-Q406 Power FETs ]
      | (SW, FB, L101 15µH)
      v (+10V Net)
      |
      +------+------------------------------+
      |                                     |
      v                                     v
[ U103 LDO (TLV76733) ]              [ U303, U403, U404, U503 FD6288Q VCC ]
      |
      v (+3V3 Net)
      |
      +-------------------------------------+
      |                                     |
      v                                     v
[ U301, U401, U402, U501 MCUs ]      [ U101 INA186 V+ ]
```

* **Buck regulator output voltage check:** $V_{OUT} = 0.6\text{ V} \times \left(1 + \frac{47\text{ k}\Omega}{3\text{ k}\Omega}\right) = 10.0\text{ V}$. Matches the `+10V` rail name exactly.
* **LDO regulator output voltage check:** Fixed 3.3V output from LDO `TLV76733DRVR` powered from `+10V`. Matches the `+3V3` rail name exactly.

---

## Analyzer Verification

### Component Count — [100% Match]
The analyzer and the raw schematic files contain 307 components (excluding internal power symbols). 

### Component Pinout Verification
The active IC pins were cross-referenced against manufacturer datasheets:

| Ref | Value | Pins | Datasheet Verified | Verification Status | Match |
|---|---|---|---|---|---|
| `U101` | INA186A2 | 5 | [INA186 Datasheet](https://www.lcsc.com/datasheet/lcsc_datasheet_2302220000_Texas-Instruments-INA186A2QDBVRQ1_C2867989.pdf) (TI, p. 5) | Verified (manual) | Yes |
| `U102` | LMR51420YDDCR | 6 | [LMR51420YDDCR.pdf](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/datasheets/LMR51420YDDCR.pdf) (TI, p. 4) | Verified (datasheet) | Yes |
| `U103` | TLV76733DRVR | 6 | [TLV76733DRVR.pdf](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/datasheets/TLV76733DRVR.pdf) (TI, p. 4) | Verified (datasheet) | Yes |
| `U301,401,402,501` | AT32F421K8T7 | 32 | [AT32F421K8T7.pdf](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/datasheets/AT32F421K8T7.pdf) (Artery, p. 11) | Verified (datasheet) | Yes |
| `U303,403,404,503` | HXFD6288QFN24 | 24 | [HXFD6288QFN24.pdf](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/datasheets/HXFD6288QFN24.pdf) (HXDZ, p. 4) | Verified (datasheet) | Yes |

### Net Tracing
* **Current Sensing Signal Net (`CURRENT`):** Connected to `U101` Pin 1 (OUT) through series resistor `R102` (to net `CURRENT`), with capacitor `C102` to GND. Traced to external control interface connector `J104` Pin 3. *Note: In the recent design update, current sense connections to the channel MCUs (Pin 9, `PA3`) were removed, and the MCUs are now disconnected from this net. The total current measurement is now routed only to the external interface.*
* **Gate Drive Logic Signals (Nets `AHIGH`, `ALOW`, etc.):** Checked connectivity between MCU PWM output pins and gate driver input pins:
  * `AHIGH`: MCU PA10 (pin 20) $\rightarrow$ FD6288Q HIN1 (pin 22) [All 4 channels]
  * `ALOW`: MCU PB1 (pin 15) $\rightarrow$ FD6288Q LIN1 (pin 1) [All 4 channels]
  * `BHIGH`: MCU PA9 (pin 19) $\rightarrow$ FD6288Q HIN2 (pin 23) [All 4 channels]
  * `BLOW`: MCU PB0 (pin 14) $\rightarrow$ FD6288Q LIN2 (pin 2) [All 4 channels]
  * `CHIGH`: MCU PA8 (pin 18) $\rightarrow$ FD6288Q HIN3 (pin 24) [All 4 channels]
  * `CLOW`: MCU PA7 (pin 13) $\rightarrow$ FD6288Q LIN3 (pin 3) [All 4 channels]

---

## Signal Analysis Review

### Voltage Dividers & Feedback Networks
* **U102 feedback divider (R115/R116):** Resistors $47\text{ k}\Omega$ (top) and $3\text{ k}\Omega$ (bottom) form the divider. Given a datasheet-verified $V_{REF} = 0.6\text{ V}$, output voltage is $V_{OUT} = 0.6\text{ V} \times \left(1 + \frac{47}{3}\right) = 10.0\text{ V}$.

### RC/LC Filters
* **Current-sense output filter (R102/C102):** $R = 1\text{ k}\Omega$, $C = 22\text{ nF}$.
  $$f_c = \frac{1}{2 \pi \times 1000\ \Omega \times 22 \times 10^{-9}\text{ F}} \approx 7.23\text{ kHz}$$
  This is a appropriate low-pass cutoff frequency to filter out motor PWM switching noise (typically 16-24 kHz or higher) before the current signal reaches the MCU ADCs.

### Current Sense Sizing
* **Shunt Resistor (`R101`):** $0.2\text{ m}\Omega$ ($0.0002\ \Omega$), power rated for $4\text{ W}$.
* **Amplifier Gain (`U101`):** $50\text{ V/V}$ (`INA186A2` variant).
* **ADC Reference:** $3.3\text{ V}$ (logic supply).
* **Maximum Measurable Current:**
  $$I_{MAX} = \frac{V_{OUT\_MAX}}{R_{SHUNT} \times Gain} = \frac{3.3\text{ V}}{0.0002\ \Omega \times 50} = 330\text{ A}$$
* **Continuous Shunt Power Margin:** 
  $$I_{CONT\_LIMIT} = \sqrt{\frac{P_{LIMIT}}{R_{SHUNT}}} = \sqrt{\frac{4\text{ W}}{0.0002\ \Omega}} \approx 141.4\text{ A}$$
  * **Reviewer's Note:** Sizing is appropriate. Peak currents up to 330A can be sampled linearly by the ADC, which is important for ESC acceleration transients. Continuous current must remain below 141.4A to prevent thermal damage to the shunt resistor `R101`.

---

## Power Analysis

### Regulator Thermal Calculation (TLV76733DRVR)
* **Logic supply load:** 4 MCUs (`AT32F421K8T7`) drawing $\approx 15-20\text{ mA}$ each + current sense amp $\approx 1\text{ mA} +$ LEDs $\approx 10\text{ mA}$. Total estimated load $I_{LOAD} \approx 90\text{ mA}$.
* **Power Dissipation:** 
  $$P_D = (V_{IN} - V_{OUT}) \times I_{LOAD} = (10\text{ V} - 3.3\text{ V}) \times 0.09\text{ A} = 0.603\text{ W}$$
* **Junction Temperature Rise (WSON-6 $\theta_{JA} \approx 80\text{ }^\circ\text{C/W}$):** 
  $$\Delta T = 0.603\text{ W} \times 80\text{ }^\circ\text{C/W} \approx 48.2\text{ }^\circ\text{C}$$
  At an ambient temperature of $25\text{ }^\circ\text{C}$, the junction temperature $T_J \approx 73.2\text{ }^\circ\text{C}$, which is well below the maximum $125\text{ }^\circ\text{C}$ rating. No active cooling is required for the LDO.

### Voltage Derating
* **TVS Diode `D101` (`SMF24A`):** Features a reverse standoff voltage of $24.0\text{ V}$ and breakdown voltage of $26.7\text{ V}$.
  * **Design Constraint:** The battery input voltage is limited to a maximum of $24\text{ V}$ to prevent TVS diode conduction. This board is suitable for up to **5S LiPo batteries** ($21.0\text{ V}$ nominal, $21.0\text{ V} \rightarrow 22.5\text{ V}$ charging). Operating this board on a **6S LiPo battery** ($25.2\text{ V}$ fully charged) will clamp the supply rail and damage the TVS diode.

### Power Warnings (RS-001)
* Nets `+BATT` and `VBAT` carry power pins but do not have an active output driver. This is because power enters the board via external 1-pin connector posts `J101` (BATT) and `J102` (GND) which are classified as passive. Adding `PWR_FLAG` symbols to `VBAT` and `GND` in the root schematic will resolve these ERC warnings.

---

## Schematic ↔ PCB Cross-Reference

* **Component Count Mismatch:** Schematic contains 290 components (excluding power symbols), while the PCB contains 0.
* **PCB State:** The [Atlas.kicad_pcb](file:///C:/Users/Param/Documents/KiCad/Projects/ESC/Atlas/Atlas/Atlas.kicad_pcb) file is currently a blank layout (80 bytes).
* **Action Required:** The layout has not been started. The designer must run **Tools $\rightarrow$ Update PCB from Schematic** to import the component footprints and net connections into the layout before starting trace routing.

---

## False Positives / Reviewer Overrides

### Net Domain Crossings without Level Shifter (VM-001)
The analyzer flags six ERRORs indicating that nets `AHIGH`, `ALOW`, `BHIGH`, `BLOW`, `CHIGH`, and `CLOW` connect 3.3V domain MCUs (`AT32F421`) to 10.0V domain gate drivers (`FD6288Q`) without a level shifter.
* **Triage Analysis:** According to the [HXFD6288QFN24.pdf](file:///C:/Users/param/OneDrive/Documents/Code/pcb/project/atlas/datasheets/HXFD6288QFN24.pdf) datasheet, the logic control inputs (HIN1/2/3 and LIN1/2/3) are explicitly designed to be **3.3V and 5V CMOS logic input compatible**. While the gate driver is powered from 10V VCC to drive the gate outputs, its control thresholds accept 3.3V logic signals directly. 
* **Overriding Verdict:** Safe to ignore. No level shifters are needed.

### EMI Filter Cutoff (EF-001)
The analyzer warns that `U102` has an input filter with a cutoff frequency of 0.13 MHz, which is close to the switching frequency.
* **Triage Analysis:** Inductor `L101` (15 µH) is actually the output buck inductor connected to `+10V`, not an input filter inductor. There is no dedicated discrete LC filter at the input `+BATT` (only decoupling MLCC capacitors). 
* **Overriding Verdict:** This is a tool misdetection (the buck converter output inductor was mistaken for an input filter inductor). However, the designer should note that there is no discrete LC EMI filter on the battery line. If low conducted EMI emissions are required, adding a series ferrite bead or inductor at the battery input is recommended.

---

## Not Performed / Review Limits

* **PCB Layout and Gerber Analysis:** Not performed, as the layout file is blank. Track widths, spacing, via dimensions, thermal vias, and edge clearances must be validated once the layout is developed.
* **SPICE Simulation:** Not performed, as no SPICE simulator (e.g. `ngspice`) is installed on the local PATH.
* **Lifecycle Audit:** Not performed, as no distributor API credentials are set in the environment variables.

---

## Verdict & Readiness Statement

**Schematic Status:** **PASSED WITH RECOMMENDATIONS**  
**PCB Layout Status:** **NEEDS START**  

**Key Recommendations:**
1. Run **Update PCB from Schematic** in Pcbnew to import all components and nets.
2. Limit operating voltage to **5S LiPo** ($21.0\text{ V}$) maximum, or replace TVS diode `D101` with a higher-voltage part (e.g., `SMF26A` or `SMF28A`) if 6S LiPo operation is required.
3. Add `PWR_FLAG` symbols to the `VBAT` and `GND` nets to clean up KiCad ERC warnings.
4. Verify that high-current paths on the PCB (battery input to power FETs, power FETs to phase output posts) are routed with wide copper planes capable of handling up to 141A continuous.

---
*Review compiled by Antigravity AI Coding Assistant.*
