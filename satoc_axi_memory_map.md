# SAToC AXI Address Map

## Why this window

NEORV32's internal address decoder claims three fixed regions: IMEM at
`IMEM_BASE` (default `0x00000000`), DMEM at `DMEM_BASE` (default `0x80000000`),
and a small internal I/O region for the built-in peripherals (UART0, GPIO,
CLINT, bootloader ROM, SYSINFO, etc.). **Every address that isn't claimed by
one of those is automatically routed out over XBUS** — that's the entire
point of XBUS: it's the CPU's "everything else" bus, bridged to AXI4 by
`xbus2axi4_bridge.vhd` inside the packaged `neorv32_vivado_ip` IP.

We get to choose where our own peripherals live inside that leftover space.
This map reserves a clean 1 MiB window starting at `0x9000_0000` — comfortably
clear of DMEM (`0x8000_0000` + a few KiB) and far from the high internal I/O
region — and gives every subsystem controller its own 64 KiB slot so the AXI
SmartConnect's address decoder has simple, non-overlapping ranges to work
with (64 KiB is far more than any of these register files actually need; it's
sized for decode simplicity and future headroom, not because each peripheral
is that large).

## Full map

| Base address | Size | Subsystem | VHDL module | Introduced |
|---|---|---|---|---|
| `0x9000_0000` | 64 KiB | OBDH register file, watchdog timer, interrupt controller | `satoc_obdh_v1_0` | Phase 6 |
| `0x9001_0000` | 64 KiB | EPS controller: register file, SPI-ADC interface, telemetry buffer | `satoc_eps_v1_0` | Phase 4 |
| `0x9002_0000` | 64 KiB | TT&C controller: TC receive path, TM transmit path | `satoc_ttc_v1_0` | Phase 5 |
| `0x9003_0000` | 64 KiB | External GPIO controller (payload/expansion I/O, separate from NEORV32's own internal GPIO used for board LEDs) | `satoc_gpio_v1_0` | Phase 3 |
| `0x9004_0000` | 64 KiB | External UART controller (RS-232 / RS-485 link, separate from NEORV32's internal UART0 debug console) | `satoc_uart_v1_0` | Phase 3 |
| `0x9005_0000` | 64 KiB | SPI controller (generic, for sensors/ADCs not already covered by EPS's dedicated SPI-ADC) | `satoc_spi_v1_0` | Phase 3 |
| `0x9006_0000` | 64 KiB | I2C controller (generic, for sensors/expansion) | `satoc_i2c_v1_0` | Phase 3 |
| `0x9007_0000` | 64 KiB | CAN 2.0B controller | `satoc_can_v1_0` | Phase 7 |
| `0x9008_0000` | 64 KiB | SpaceWire controller (DS encode/decode) | `satoc_spw_v1_0` | Phase 7 |
| `0x9009_0000`–`0x900E_FFFF` | 384 KiB | **Reserved** — payload / future expansion | — | — |
| `0x900F_0000` | 64 KiB | Phase 2 AXI bus-routing test peripheral (temporary — remove from the design once Phase 3 lands) | `satoc_bus_test_v1_0` | Phase 2 |

## Phase 2 test peripheral register layout

`satoc_bus_test_v1_0` at base `0x900F_0000`, 4 word-aligned 32-bit read/write
registers, no side effects (pure scratch storage — good for proving the bus
works before any real peripheral logic exists):

| Offset | Register | Access |
|---|---|---|
| `0x0` | SCRATCH0 | R/W |
| `0x4` | SCRATCH1 | R/W |
| `0x8` | SCRATCH2 | R/W |
| `0xC` | SCRATCH3 | R/W |

## RS-485 / RS-232 / SPI / I2C / GPIO physical routing note

The controllers at `0x9003_0000`–`0x9006_0000` are the AXI-side register
interfaces only. Which physical FPGA pins they drive (PMOD/Arduino headers
on the PYNQ-Z2) will be assigned in their XDC constraints when each is built
in Phase 3 — this document only fixes the *bus-side* addresses so firmware
written now (like the Phase 2 bus test) won't need to change later.
