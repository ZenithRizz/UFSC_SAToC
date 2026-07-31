# SAToC Architecture Proposal

Assignment task: *"Propose a Satellite-on-a-Chip architecture showing
how the OBDH, TT&C, EPS, memory resources, and the RISC-V processor are
integrated within the FPGA"* and *"Develop a detailed block diagram of the
proposed architecture."*

## 1. Platform

**Xilinx Zynq XC7Z020 (Digilent PYNQ-Z2)** — available at both LNMIIT and
SpaceLab(not confirmed), satisfying the assignment's preference for a shared development
platform. The design uses the **programmable logic (PL) only**; the Zynq's
hardened ARM processing system (PS7) is deliberately left unused so that the
entire OBC is a soft RISC-V core implemented in fabric — this is what makes it
a genuine *Satellite-on-a-Chip* rather than a Zynq PS/PL hybrid design, and it
keeps the architecture portable to a non-Zynq, PL-only FPGA later if needed.

## 2. Integration model: one CPU, one bus, one address map

All three subsystems — OBDH, EPS, TT&C — plus the communication controllers
(GPIO, UART, SPI, I2C, CAN, SpaceWire) are **memory-mapped AXI4-Lite
peripherals** hanging off a single AXI SmartConnect interconnect, which is in
turn driven by a single AXI4 master: the NEORV32 RISC-V core. There is
deliberately no separate microcontroller per subsystem — one CPU running the
OBDH firmware is the single point of orchestration, matching how the
real GOLDS-UFSC satellite works today: the CDR documentation for
GOLDS-UFSC describes exactly this pattern in PCB form (separate EPS, OBDH, and
TTC *modules* exchanging housekeeping data over SPI/I2C/UART links), and this
project's job is to collapse that same functional split onto one chip, not to
invent a new architecture from scratch.

Memory resources split into two tiers:

- **Internal to the CPU** (not on the AXI bus at all): NEORV32's own IMEM
  (16 KiB instructions), DMEM (8 KiB data), and bootloader ROM, reached over
  NEORV32's internal bus — this is where the OBDH firmware itself lives and
  runs from.
- **On the AXI bus**: every subsystem's register file (status, control,
  telemetry buffers) — not bulk storage, just the memory-mapped interface
  each subsystem controller exposes to the CPU.

## 3. Block diagram

```mermaid
flowchart TB
    subgraph FPGA["Xilinx Zynq XC7Z020 — programmable logic only"]
        CPU["NEORV32 RISC-V Core<br/>rv32imc, runs OBDH firmware"]
        MEM["Internal IMEM / DMEM<br/>+ bootloader ROM"]
        BRIDGE["XBUS to AXI4 Bridge"]
        UART0["UART0 debug console"]
        AXI["AXI SmartConnect<br/>0x9000_0000 - 0x900F_FFFF"]

        OBDH["OBDH controller<br/>reg file + WDT + IRQ (Phase 6)"]
        EPS["EPS controller<br/>SPI-ADC + TM buffer (Phase 4)"]
        TTC["TT&amp;C controller<br/>TC rx / TM tx (Phase 5)"]
        PERIPH["GPIO / UART / SPI / I2C<br/>(Phase 3 - done)"]
        CAN["CAN 2.0B controller (Phase 7)"]
        SPW["SpaceWire controller (Phase 7)"]

        CPU --- MEM
        CPU --> BRIDGE
        CPU --> UART0
        BRIDGE --> AXI
        AXI --> OBDH
        AXI --> EPS
        AXI --> TTC
        AXI --> PERIPH
        AXI --> CAN
        AXI --> SPW
    end

    EXTUSB["USB-UART adapter<br/>(bring-up console)"]
    EXTPWR["EPS power conversion HW<br/>(external, per assignment)"]
    EXTRF["Radio front-end / antenna<br/>(external, per assignment)"]
    EXTCAN["CAN bus nodes"]
    EXTSPW["SpaceWire link"]
    EXTSENS["Sensors / EDCs / payloads<br/>via GPIO, SPI, I2C"]

    UART0 --> EXTUSB
    EPS --> EXTPWR
    TTC --> EXTRF
    CAN --> EXTCAN
    SPW --> EXTSPW
    PERIPH --> EXTSENS
```

(Rendered as an SVG in the chat above; this Mermaid version is the one that
displays directly on GitHub, per the assignment's repository deliverable.)

## 4. Data flow (firmware's view)

This mirrors the GOLDS-UFSC CDR's documented OBDH/EPS/TTC operational cycle,
translated from "three PCB modules talking over UART/SPI/I2C" to "three
memory-mapped register blocks on one AXI bus":

1. **Boot**: NEORV32 boots from its internal bootloader ROM, runs the OBDH
   housekeeping loop from IMEM/DMEM.
2. **EPS polling**: OBDH firmware reads the EPS controller's registers
   (Phase 4) at a fixed housekeeping rate — voltage/current/status — the same
   role the real EPS 2.0 module's periodic report plays in the CDR design.
3. **TT&C servicing**: OBDH firmware checks the TT&C controller (Phase 5) for
   an incoming telecommand each cycle, and periodically builds a telemetry
   frame from EPS + OBDH housekeeping data for the TT&C controller to
   transmit — again matching the CDR's "TTC receives a simplified version,
   OBDH receives a complete version" split, just implemented as two register
   views instead of two physical links.
4. **Watchdog**: OBDH controller's hardware watchdog (Phase 6) is kicked once
   per loop; a missed kick resets the CPU, standing in for the same
   reliability role the CDR's boot/reset sequencing describes.
5. **Peripheral I/O**: GPIO/UART/SPI/I2C (Phase 3, complete) and CAN/SpaceWire
   (Phase 7) are available to any subsystem controller or directly to OBDH
   firmware for sensor/actuator/payload access, exactly as the assignment's
   "auxiliary protocols for peripherals and sensors" requirement asks for.

## 5. Address map

Full detail in `satoc_axi_memory_map.md` (Phase 2 deliverable); summary:

| Base | Subsystem |
|---|---|
| 0x9000_0000 | OBDH |
| 0x9001_0000 | EPS |
| 0x9002_0000 | TT&C |
| 0x9003_0000–0x9006_0000 | GPIO / UART / SPI / I2C |
| 0x9007_0000 | CAN |
| 0x9008_0000 | SpaceWire |

## 6. Why this satisfies the assignment's architecture requirement

- **Single FPGA integration** of OBDH + TT&C + EPS + RISC-V + memory: shown
  above, all on one AXI bus inside one XC7Z020.
- **External-hardware boundary respected**: EPS power conversion and RF/antenna
  stages are explicitly kept off-chip (dashed boxes), matching the
  assignment's "some components cannot be implemented directly in FPGA logic"
  scoping note.
- **Grounded in SpaceLab heritage**: the OBDH/EPS/TT&C data-flow roles are
  taken directly from the GOLDS-UFSC CDR's documented module behavior, not
  invented — this is a single-chip re-implementation of a design SpaceLab
  already flies, not a from-scratch architecture.
- **Traceable to the phase-by-phase build**: every box in the diagram maps to
  a specific phase already completed (1–3) or planned (4–8) in this project,
  so the architecture document and the actual repository stay in sync.
