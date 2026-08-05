# SAToC — Satellite-on-a-Chip

Satellite-on-a-Chip (SAToC) consolidates a nanosatellite's OBDH, TT&C, and EPS
service-platform functions, plus their communication interfaces, onto a
single FPGA. Instead of separate boards for each subsystem, everything is
built as memory-mapped AXI4-Lite peripherals on one AXI bus, orchestrated by
a single NEORV32 RISC-V core implemented entirely in programmable logic.

Components that can't be implemented in FPGA logic, such as EPS power
conversion stages and RF front-ends, stay off-chip; everything else runs on
the same die as the processor.

**Target platform:** Digilent PYNQ-Z2 (Xilinx Zynq XC7Z020), programmable
logic only, no PS7
**Processor:** NEORV32 (RISC-V, rv32imc)
**Status:** Phases 0–3 complete (repo setup, NEORV32 bring-up, AXI
interconnect, GPIO/UART/SPI/I2C controllers); EPS, TT&C, OBDH, and
CAN/SpaceWire controllers in progress

---SpaceLab UFSC internship project---
