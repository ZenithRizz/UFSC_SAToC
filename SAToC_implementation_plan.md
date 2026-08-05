# SAToC Implementation Plan

Target platform: Xilinx Zynq XC7Z020 (PYNQ-Z2). Toolchain: Vivado + RISC-V GCC.

| Phase | Scope |
|---|---|
| 0 | Project setup: create the SpaceLab GitHub repository, `hdl/` `sim/` `sw/` `docs/` folder structure, `.gitignore` for Vivado run/cache directories, push everything produced so far |
| 1 | NEORV32 bring-up: RTL-only top level, PYNQ-Z2 pin constraints, boot firmware, UART "hello world" over a USB-UART adapter |
| 2 | AXI interconnect: package NEORV32 as a Vivado AXI4 master IP, build the block design, define the full AXI address map, prove bus routing with a test peripheral |
| 3 | Peripheral controllers: AXI4-Lite GPIO, UART, SPI, and I2C, each with a synthesizable core and a self-checking loopback testbench |
| 4 | EPS controller: AXI4-Lite register file, SPI-ADC interface for simulated voltage/current readings, telemetry buffer, testbench, C firmware driver |
| 5 | TT&C controller: telecommand receive path, telemetry transmit path, simplified CCSDS-style frame (sync + header + data + CRC-16), testbench, firmware driver |
| 6 | OBDH controller: register file, watchdog timer, interrupt controller, plus the housekeeping firmware loop (poll EPS, check TC, build/send TM, kick watchdog) |
| 7 | CAN 2.0B and SpaceWire controllers, each with a loopback testbench |
| 8 | System integration: final top-level tying every subsystem together, system-level testbench, full hardware demo on the PYNQ-Z2 |

## Documentation deliverables

Here's where things stand against the assignment's documentation requirements:

| Document | Status |
|---|---|
| Schedule / implementation plan (this file) | Done |
| Architecture + block diagram (`SATOC_ARCHITECTURE.md`) | Done |
| AXI memory map (`satoc_axi_memory_map.md`) | Done |
| Platform evaluation write-up (why PYNQ-Z2, vs. what alternatives at LNMIIT/SpaceLab) | Not started |
| Architecture / data-flow flowcharts (separate from the block diagram) | Not started |
| Test Plan document | Not started |
| Simulation results (captured logs/screenshots per testbench) | Not started |
| FPGA utilization report | Not started |

## Deliverables produced so far

- HDL: NEORV32 top wrapper, AXI test peripheral, GPIO/UART/SPI/I2C controllers.
- Testbenches: one self-checking loopback test per Phase 3 controller.
- Documentation: `PHASE1_GUIDE.md`, `PHASE2_GUIDE.md`, `PHASE3_GUIDE.md`,
  `satoc_axi_memory_map.md`, `SATOC_ARCHITECTURE.md` (this project's block
  diagram and design rationale).
- Firmware: boot test (`main.c`), AXI bus-routing test (`bus_test.c`).

## Open items to resolve before Phase 4

A few things carried over from the architecture review that are worth a
professor check-in before Phase 4 starts: interrupt-driven vs. polled
peripheral servicing, bus-timeout/error-response policy, the target EPS ADC
part (matters for I2C clock-stretching compatibility), and how much of
SpaceWire's physical layer to actually implement given the PYNQ-Z2's
single-ended headers.
