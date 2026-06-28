# Custom FSM-Based UART IP

Full-duplex UART controller implemented in Verilog, targeted at the **Intel MAX 10 (10M50DAF484C7G)** FPGA. Both the transmit and receive paths are built from discrete RTL modules — separate FIFOs, shift registers, parity checkers, and FSMs — with all baud clocking derived from a single on-chip PLL.

---

## Architecture

```
sys_clk (50 MHz)
    │
    ▼
 pll_baud  ──► c0 → baud_clk16  (153.6 kHz, 16× oversampling clock for RX FSM)
    │      ──► c1 → baud9600    (9600 Hz,  bit-rate clock for TX and RX shift registers)
    │
    ├── tx_uart ──────────────────────────────────────► TX_OUT
    │     ├── tx_fifo          (async CDC FIFO, depth 8, Gray-coded pointers)
    │     ├── tx_shift_register (PISO, 11-bit frame: START | D0–D7 | PARITY | STOP)
    │     ├── tx_parity        (even parity, XOR reduction over 8 data bits)
    │     └── tx_fsm           (IDLE → SHIFT, 2-state, count to 11)
    │
    └── rx_uart ◄───────────────────────────────────── RX_IN
          ├── rx_shift_register (SIPO, 11-bit, reset-to-ones, MSB = first_bit)
          ├── rx_parity        (even parity check, XOR over bits [9:2])
          ├── rx_fsm           (IDLE → DATA → PARITY → STOP, 16× oversampling)
          ├── pulse16          (single-cycle FIFO write-enable stretcher, 15-cycle hold)
          └── rx_fifo          (async CDC FIFO, depth 8, Gray-coded pointers)
          
```

The top-level `uart.v` wires the two paths together in loopback: received bytes from the RX FIFO are directly fed into the TX FIFO (`tx_fifo_in = rx_fifo_out`), so the device echoes every received byte.

---

## Clock Plan

| PLL Output | Signal | Frequency | Used by |
|---|---|---|---|
| `c0` | `baud_clk16` | **153.6 kHz** | `rx_fsm` sample counter (16× oversampling) |
| `c1` | `baud9600` | **9.6 kHz** | TX shift register, TX FIFO read, RX shift register |

PLL input: 50 MHz system clock (`inclk0_input_frequency = 20000 ps`).
Multiplier/divider from `pll_baud.v`: ×48/÷15625 for c0, ×3/÷15625 for c1.

---

## Module Reference

### `uart.v` — Top Level

| Port | Dir | Width | Description |
|---|---|---|---|
| `sys_clk` | IN | 1 | 50 MHz system clock (PIN P11) |
| `rst` | IN | 1 | Active-high synchronous reset (PIN C10) |
| `rx_in` | IN | 1 | Serial receive line (PIN V9) |
| `rx_ready` | IN | 1 | Assert to read next byte from RX FIFO (PIN C11) |
| `tx_fifo_en` | IN | 1 | Assert to write a byte to TX FIFO (PIN D12) |
| `tx_out` | OUT | 1 | Serial transmit line (PIN W10) |
| `rx_fifo_empty` | OUT | 1 | High when RX FIFO has no data (PIN V10) |
| `tx_fifo_full` | OUT | 1 | High when TX FIFO cannot accept more data |

Internal wiring: `tx_wr_en = ~tx_fifo_full & tx_fifo_en`, `rx_rd_en = ~rx_fifo_empty & rx_ready`.

---

### TX Path — `tx_uart/`

#### `tx_fifo.v`
Asynchronous FIFO (depth 8, data width 8). Write clock = `sys_clk`, read clock = `baud9600`. Gray-coded read/write pointers for safe CDC. Full/empty flags are combinationally generated.

#### `tx_shift_register.v`
11-bit parallel-in serial-out register. On `tx_load_en`: loads frame as `{STOP=1, PARITY, D7..D0, START=0}` into `q[10:0]`. On shift: outputs `q[0]` to `tx_out` and shifts in `1'b1` from the MSB end.

#### `tx_parity.v`
Even parity generator: `parity_check = ^din[7:0]`.

#### `tx_fsm.v`
Two-state FSM clocked on `baud9600`.

- **IDLE**: asserts `tx_load_en`, waits for `tx_start` (`= ~fifo_empty & ~tx_busy`).
- **SHIFT**: clears `tx_load_en`, counts 11 baud ticks (bits 0–10), then returns to IDLE.

#### `tx_uart.v`
Wrapper that instantiates the four TX submodules and connects them. `tx_start` is derived internally as `~txfifo_empty & ~tx_busy`.

---

### RX Path — `rx_uart/`

#### `rx_shift_register.v`
11-bit serial-in parallel-out register. Clocked on `baud9600` (bit rate). Shifts MSB-first: `q <= {data_in, q[10:1]}`. Resets to all-ones. `first_bit = data_out[10]` (the most recently received bit, used by the FSM for start/stop detection).

#### `rx_parity.v`
Even parity checker: `rx_parity_check = ^din[9:2]` (checks the 8 data bits within the 11-bit shift register frame).

#### `rx_fsm.v`
Four-state FSM clocked on `baud_clk16` (16× oversampling). Uses a 4-bit `sample_counter` to find bit centres.

| State | Transition condition |
|---|---|
| **IDLE** | Falls to DATA after 8 consecutive samples of `rx_in = 0` (start-bit validation) |
| **DATA** | Samples each bit at count=15; advances `count` 0→7; moves to PARITY at count=7 |
| **PARITY** | Samples at count=15; checks `rx_in == rx_parity_in`; moves to STOP either way |
| **STOP** | Samples at count=15; if `rx_in = 1` (valid stop), latches `data_fsm[8:1]` into `temp_data` and asserts `fifo_wren` (if parity passed); returns to IDLE |

#### `pulse16.v`
Converts the single-cycle `fifo_wren` pulse from `rx_fsm` into a 15-cycle wide pulse on `baud_clk16`, giving the `rx_fifo` write strobe enough width to be captured reliably across the clock domain boundary.

#### `rx_fifo.v`
Asynchronous FIFO (depth 8, data width 8). Write clock = `baud9600` (negedge), read clock = `sys_clk`. Also exposes a `read_ack` register that goes high for one `sys_clk` cycle after a successful read.

#### `rx_uart.v`
Wrapper that instantiates all five RX submodules. Note: `rx_baud_generator.v` is present in the source tree but is **not instantiated** in `rx_uart.v`; both baud clocks come from `pll_baud` at the top level.

---

## Resource Utilisation (from Fitter, Feb 2025)

| Resource | Used | Available | % |
|---|---|---|---|
| Logic elements | 321 | 49,760 | < 1% |
| Combinational functions | 228 | 49,760 | < 1% |
| Dedicated registers | 243 | 49,760 | < 1% |
| I/O pins | 8 | 360 | 2% |
| PLLs | 1 | 4 | 25% |
| Memory bits | 0 | 1,677,312 | 0% |

---

## Timing Notes

No `.sdc` constraints file was included in this compilation. The Timing Analyzer auto-derived clocks from the PLL. Results from the Slow 1200 mV 85 °C corner:

- **`baud9600` (c1) setup slack: −0.666 ns** — timing violation on this domain. This is expected without a proper SDC; the 9.6 kHz clock has an extremely long period so the violation is likely an analyser artefact, but an SDC should be added to confirm.
- `sys_clk` setup slack: +1.224 ns — clean.
- `baud_clk16` (c0) setup slack: +6506 ns — well within budget.

**To do:** add `uart.sdc` with `create_clock`, `create_generated_clock` (or `derive_pll_clocks`), and appropriate `set_false_path` / `set_max_delay` constraints for the CDC crossings between `sys_clk` and `baud9600`.

---

## Known Issues

- `txfifo_empty` in `tx_uart.v` is declared as a wire but has no driver — Quartus defaults it to 0 (TX FIFO always appears non-empty). This is a connectivity warning flagged by the compiler.
- Implicit nets `tx_wr_en` and `rx_rd_en` in `uart.v` — should be explicitly declared.
- 32-to-4-bit truncation warnings in both FIFOs (`tx_fifo.v` lines 43/55, `rx_fifo.v` lines 44/58) — the pointer arithmetic produces a 32-bit result assigned to a 4-bit Gray register. Needs an explicit width cast.
- `rx_baud_generator.v` is compiled but unused; it was replaced by the PLL output.

---

## Repository Structure

```
uart/
├── uart.v                      # Top-level wrapper (echo loopback)
├── pll_baud.v / .qip / .ppf   # ALTPLL megafunction (50 MHz → 153.6 kHz, 9.6 kHz)
├── tx_uart/
│   ├── tx_uart.v               # TX subsystem wrapper
│   ├── tx_fifo.v               # Async TX FIFO (depth 8)
│   ├── tx_shift_register.v     # PISO shift register (11-bit UART frame)
│   ├── tx_parity.v             # Even parity generator
│   └── tx_fsm.v                # TX control FSM (IDLE / SHIFT)
├── rx_uart/
│   ├── rx_uart.v               # RX subsystem wrapper
│   ├── rx_shift_register.v     # SIPO shift register (11-bit)
│   ├── rx_parity.v             # Even parity checker
│   ├── rx_fsm.v                # RX control FSM (IDLE/DATA/PARITY/STOP, 16× OS)
│   ├── pulse16.v               # FIFO write-enable pulse stretcher
│   ├── rx_fifo.v               # Async RX FIFO (depth 8)
│   └── rx_baud_generator.v     # Unused; baud clocks come from pll_baud
├── uart.qpf / uart.qsf         # Quartus Prime project files
├── output_files/               # Fitter/STA/assembler reports and .sof bitstream
└── docs/
    └── blockdgrm_fsm_uart.png  # Architecture block diagram
```

---

## Opening in Quartus Prime

```
# Quartus Prime 23.1 Lite
File → Open Project → uart.qpf
Processing → Start Compilation
```

To program the device after compilation:

```
Tools → Programmer → Add File → output_files/uart.sof → Start
```

Target board: any MAX 10 board with a 50 MHz oscillator on the clock pin assigned to `sys_clk` (PIN P11 in the current `.qsf`).
