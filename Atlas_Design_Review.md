# Atlas ESC Design Review & Critical Audit

**Project:** Atlas Multi-Channel ESC (KiCad 10.0, 5 hierarchical sheets: 1 root + 4 instanced Motor sheets, no PCB layout started yet)  
**Date:** June 22, 2026  
**Auditor:** Antigravity AI Coding Assistant  
**Schematic Status:** **FAILED (CRITICAL REVISION REQUIRED)**  

---

## Executive Summary

A follow-up audit of the schematic design files on disk (`Atlas.kicad_sch` and `VegaESC.kicad_sch`) has verified that **three major issues have been successfully resolved**:
1. **MOSFET Pin Mismatches:** All 24 power MOSFET symbols are now correctly mapped to the 5-pin `NTMFS5C430NLT1G` symbol.
2. **MCU Footprints:** All 4 channel MCUs have been updated to the QFN-28 `AT32F421G8U7`.
3. **Channel Isolation:** All sheet-specific control, telemetry, BEMF, and drive nets are successfully isolated as local labels, preventing channel-to-channel short circuits.

However, **five critical design issues remain outstanding** in the files on disk and must be corrected before pushing the netlist to the PCB Editor.

---

## 1. Pinout & Logic Mapping

### ✅ RESOLVED: Sheet-Specific Net Shorts
*   All channel-specific control, BEMF sensing, and motor output nets (such as `AHIGH`/`ALOW`, `BEMFx`, `MOTORA/B/C`, `GLx`/`GHx`, and SWD programming lines `SWDIO`/`SWDCLK`) have been successfully converted from global to local labels. They are now correctly isolated and namespaced per channel sheet (e.g. `/Motor_1/AHIGH` vs `/Motor_2/AHIGH`).

### ⚠️ WARNING: Shared Reset (NRST)
*   **Current State:** The reset pins (`Pin 4, NRST`) of all four MCUs (`U203`, `U303`, `U403`, `U503`) remain tied to the same global `NRST` net. 
*   **Risk:** If any individual MCU experiences a local brownout or reset condition, it will pull the line low and reset *all* four MCUs simultaneously, which is a flight safety risk.
*   **Action Required:** Change `NRST` to a local label inside `VegaESC.kicad_sch` to isolate MCU resets, or verify that a global reset behavior is desired for the system.

---

## 2. Power Delivery & Decoupling

### 🛑 CRITICAL: Bootstrap Diodes Voltage Rating Too Low (Outstanding)
*   **Current State:** Diodes `D207–D209`, `D307–D309`, `D407–D409`, `D507–D509` are still labeled as Schottky diodes `1N5819WS` rated for **40V**.
*   **Risk:** During high-speed switching on 5S/6S LiPo, the reverse voltage across these diodes will experience inductive voltage spikes that easily exceed 40V. This will cause the diodes to fail short, shorting the `VBx` bootstrap node directly to the `+10V` rail, destroying the gate drivers (`U202`), the 10V regulator (`U101`), and downstream logic.
*   **Action Required:** Update the value of all 12 bootstrap diodes in the schematic to a **100V ultra-fast recovery or Schottky diode** (e.g., `1N4148WS`, `B0560W`, or `MURA110`).

### TVS Diode Voltage Derating
*   **Current State:** TVS diodes `D103` and `D104` on the battery input are `SMF24A` (24V reverse standoff voltage, 26.7V breakdown).
*   **Constraint:** This limits the board to a maximum of **5S LiPo batteries** (21V nominal, 22.5V charged). Operating this board on a **6S LiPo battery** (25.2V charged) will clamp the supply rail, causing the TVS diodes to conduct, overheat, and fail short.
*   **Action Required:** If 6S LiPo operation is required, replace `D103` and `D104` with a 26V or 28V TVS diode (e.g., `SMF26A` or `SMF28A`).

---

## 3. Footprint & PCBA Readiness

### ✅ RESOLVED: MOSFET Pin Mismatch (All 24 FETs)
*   All power MOSFETs (`Q201–Q206`, `Q301–Q306`, `Q401–Q406`, `Q501–Q506`) have been successfully updated to the 5-pin symbol matching the physical DFN-5/SO-8FL package pinout (`Gate=Pin 4`, `Drain=Pin 5`, `Source=Pins 1, 2, 3` in parallel). Pins 1, 2, and 3 are correctly tied to GND for low-side FETs and to the phase nodes for high-side FETs.

### 🛑 CRITICAL: Buck Regulator Pinout Cache Issue (U101)
*   **Current State:** The buck regulator `U101` (`LMR51420YDDCR`) symbol on disk still shows `GND` on Pin 1, `SW` on Pin 2, `VIN` on Pin 3, `FB` on Pin 4, `EN` on Pin 5, and `CB` on Pin 6.
*   **Risk:** The physical SOT-23-6 (DDCR) package uses: `1=CB, 2=GND, 3=FB, 4=EN, 5=VIN, 6=SW`. This connects GND to CB, CB to SW, and VIN to FB, leading to immediate destruction of the regulator on power-up.
*   **Why it is Outstanding:** If you updated the symbol in your global library or via `easyeda2kicad`, you **must** run **Tools ➔ Update Symbols from Library** in the KiCad schematic editor for the changes to propagate to the schematic file on disk. Currently, the file still contains the old cached symbol.

### 🛑 CRITICAL: 3.3V LDO Regulator Pin Swap (U102)
*   **Current State:** LDO `U102` (`TLV76733DRVR`) remains mapped as: Pin 1 = OUT, Pin 2 = FB/SNS, Pin 3 = GND, Pin 4 = EN, Pin 5 = GND, Pin 6 = IN.
*   **Risk:** The physical WSON-6 package of the `TLV76733DRVR` has: `1=IN, 2=GND, 3=EN, 4=NC, 5=OUT, 6=IN`. This shorts OUT (Pin 5) to the GND plane, shorts GND (Pin 2) to the output, and disables the chip (Pin 3 tied to GND).
*   **Action Required:** Update the schematic symbol for `U102` to use the correct WSON-6 pin mapping: 1=IN, 2=GND, 3=EN, 4=NC, 5=OUT, 6=IN, 7(EP)=GND.

### 🛑 CRITICAL: Current Sense Amplifier Pin Swap (U103)
*   **Current State:** Current sense amplifier `U103` (`INA186A2`) remains mapped as Pin 1 = GND and Pin 2 = OUT.
*   **Risk:** The physical SOT-23-5 (DBV) package of the `INA186` has Pin 1 = OUT and Pin 2 = GND. This shorts the OUT signal to GND and GND to the output.
*   **Action Required:** Update the symbol pinout for `U103` to swap Pin 1 and Pin 2 (Pin 1 = OUT, Pin 2 = GND).

---

## 4. High-Current Routing Prep

### 🛑 CRITICAL: Missing Kelvin Sense Resistors and Noise Filter (U103)
*   **Current State:** The input pins of current sense amplifier `U103` (`Pin 4, IN-` and `Pin 5, IN+`) are still connected directly to the high-current nets `+BATT` and `VBAT` with no series resistors.
*   **Risk:** KiCad will merge the sense traces with the main power planes during layout, preventing separate Kelvin differential pair routing. High-current switching noise on the power plane will corrupt the differential voltage reading.
*   **Action Required:**
    1.  Insert two series resistors (e.g., `R107` and `R108`, value 10Ω to 100Ω) in the sense lines right next to the pads of shunt resistor `R105`.
    2.  Rename the nets on the IC side of these resistors to `ISENSE_N` and `ISENSE_P` (connected to `U103` Pin 4 and Pin 5 respectively) to isolate them.
    3.  Add a differential capacitor (e.g., `C122`, 22nF) between `ISENSE_P` and `ISENSE_N` right next to the pins of `U103` to form an RC input filter to suppress switching noise.

---

## Action Plan & Required Corrections

```mermaid
graph TD
    A[Identify Remaining Schematic Errors] --> B[Update Symbol Mappings from Library]
    A --> C[Swap U102/U103 Pin Mappings]
    A --> D[Add Sense Filter & New Nets]
    A --> E[Increase Diode Ratings]
    
    B --> B1[Run Tools -> Update Symbols from Library for U101 LMR51420]
    C --> C1[U102: 1=IN, 2=GND, 3=EN, 4=NC, 5=OUT, 6=IN, 7(EP)=GND]
    C --> C2[U103: Swap Pin 1 (OUT) and Pin 2 (GND)]
    
    D --> D1[Insert R107/R108 next to R105]
    D --> D2[Rename nets to ISENSE_P / ISENSE_N]
    D --> D3[Add C122 across ISENSE_P / ISENSE_N]
    
    E --> E1[Change D207-D509 bootstrap diodes to 100V rated parts (e.g. 1N4148WS)]
```

---
*Review compiled by Antigravity AI Coding Assistant.*
