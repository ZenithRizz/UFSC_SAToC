# SAToC Architectural Rationale

## 1. Processing Core: Single Core vs. Multi-Core Subsystems

- **Decision:** One central NEORV32 RISC-V core acts as the sole AXI4 master.
- **Why not a dedicated core per subsystem?**

- **LUT/Resource Sparing:** The target FPGA has a limited resource budget (~53,200 LUTs). Tripling CPU cores burns logic blocks on redundant execution units.
- **Elimination of Inter-Core Bottlenecks:** Multiple cores require complex inter-processor communication (IPC), shared memory arbitration, and synchronization primitives.
- **System Alignment:** The original flight-proven GOLDS-UFSC On-Board Computer (OBC) operates on a single central controller polling peripheral systems.

## 2. Interconnect & Protocol Choice: AXI4 / AXI4-Lite

- **Decision:** Full AXI4 for the CPU master port, AXI4-Lite for peripheral slaves via AXI SmartConnect.
- **Why not Wishbone or a custom point-to-point bus?**

- **Toolchain Integration:** Staying strictly within AXI enables Vivado’s automated Address Editor, memory-map generation, and AXI SmartConnect crossbar generation.
- **Full AXI4 vs. AXI4-Lite Separation:** The CPU uses full AXI4 to allow multi-word burst transactions (necessary for high speed block transfers like Flash memory access). Subsystems (EPS, TT&C, OBDH) only require basic Command/Status registers, so implementing full AXI4 burst logic inside those modules would waste gate count with zero functional gain.

## 3. CPU Interface: Native Bus vs. XBUS-to-AXI4 Bridge

- **Decision:** Retain NEORV32's native internal XBUS and use a vendor-supported wrapper bridge to interface with AXI4.
- **Why not modify NEORV32's internal RTL to natively output AXI4?**

- **Maintainability & Verification:** Modifying internal execution pipelines or core bus protocols breaks upstream updates and requires extensive formal re-verification of the CPU core. The wrapper bridge isolates processor internals from the SoC fabric cleanly.

## 4. Memory Layout: Tightly-Coupled (TCM) vs. Shared AXI Memory

- **Decision:** Keep Instruction Memory (IMEM) and Data Memory (DMEM) tightly coupled to the CPU's internal XBUS , outside the main AXI interconnect.
- **Why couldn't IMEM/DMEM sit on the AXI bus alongside peripherals?**

- **Real-Time Determinism:** Instruction fetches happen on almost every clock cycle. If program memory reads had to contend with peripheral bus arbitration (for example, EPS reading an ADC or TT&C handling a packet burst), instruction execution timing would stall dynamically, violating deterministic execution guarantees required for spaceflight watchdogs.

## 5. Peripheral Isolation: Independent Slaves vs. Unified Register Block

- **Decision:** Segment EPS, TT&C, and OBDH into individual AXI4-Lite slave modules.
- **Why not a single unified memory-mapped register file?**

- **Spatial Fault Isolation:** In a unified block, a pointer bug or illegal memory write from firmware could corrupt adjacent registers (for example, an EPS configuration error inadvertently clearing a watchdog in OBDH). Hardware-decoded address boundaries prevent cross-subsystem memory corruption.
- **Modular Verification:** Separate AXI interfaces allow independent HDL development, isolated testbench verification, and explicit utilization reporting required for design reviews.

## 6. Address Map Placement

- **Decision:** Base peripheral addresses start at `0x9000_0000` with 64 KB allocated per slot.
- **Why `0x9000_0000`?**

- **CPU Reserved Memory Clearing:** NEORV32 hard-reserves lower address regions:

- `0x0000_0000`: Internal IMEM
- `0x8000_0000`: Internal DMEM
- High memory spaces: Internal I/O registers (UART, Bootloader ROM, CLINT)
- `0x9000_0000` is the first cleanly aligned, unassigned address range that clears all internal CPU structures.
- **Why 64 KB allocations when peripherals only use about 32 bytes?**

- **Decoder Efficiency:** AXI address decoders operate most efficiently on power-of-two address blocks.
- **Firmware Stability:** Generous spacing allows peripheral features such as hardware packet buffers or added telemetry channels to expand in future hardware revisions without modifying base firmware offset pointers.

## 7. Soft Core Selection: RISC-V vs. Hardened ARM Core (PS7)

- **Decision:** Soft NEORV32 RISC-V core instantiated in FPGA logic.
- **Why not use the built-in Zynq PS7 hard ARM core?**

- **Target ASIC Portability:** Hardened cores inside commercial FPGAs lock the system to vendor silicon. Soft RISC-V IP cores are vendor-neutral and can be directly re-synthesized for radiation-hardened ASIC processes such as TSMC space-qualified nodes or Microchip rad-hard FPGAs for flight missions.

## Memory Allocation & Decoding Map

| Subsystem Module | Base Address | End Address | Address Region Size | Interface Type |
|---|---|---|---|---|
| CPU Internal IMEM | `0x0000_0000` | Dynamic | System dependent | Tightly-Coupled XBUS |
| CPU Internal DMEM | `0x8000_0000` | Dynamic | System dependent | Tightly-Coupled XBUS |
| EPS Controller | `0x9000_0000` | `0x9000_FFFF` | 64 KB | AXI4-Lite |
| TT&C Controller | `0x9001_0000` | `0x9001_FFFF` | 64 KB | AXI4-Lite |
| OBDH Peripheral Block | `0x9002_0000` | `0x9002_FFFF` | 64 KB | AXI4-Lite |
| Flash Interface | `0x9003_0000` | `0x9003_FFFF` | 64 KB | AXI4-Lite |
