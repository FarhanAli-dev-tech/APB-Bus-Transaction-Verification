# APB Bus Transaction-Level Verification (SystemVerilog)

A layered, class-based (OOP) UVM-style testbench for verifying a simple **APB (Advanced Peripheral Bus) Slave** memory device — built using SystemVerilog with `mailbox`/`event` based synchronization (no UVM library dependency).

---

## 📌 Overview

This project implements a classic **Generator → Driver → Monitor → Scoreboard** layered verification architecture to verify an APB slave that models a 256-word memory. The testbench performs **10 random writes** followed by **10 reads to the same addresses**, and checks read data against a reference memory model. Two reads are intentionally corrupted by the monitor to demonstrate that the scoreboard correctly detects mismatches.

---

## 🏗️ Testbench Architecture

<img width="627" height="470" alt="image" src="https://github.com/user-attachments/assets/afb12f69-2298-4a85-9fb2-8b9cc0bc6a85" />

**Flow:**
- **Generator** — randomizes 10 write transactions, saves their addresses, then randomizes 10 read transactions reusing those exact addresses. Pushes transactions into `mbx_gen_drv`.
- **Driver** — pulls transactions from the mailbox and drives them onto the `apb_if` interface following the APB protocol (SETUP → ACCESS → wait for `pready`). Signals `drv_done` after each transaction completes.
- **Monitor** — passively watches the interface, captures completed transactions (`psel && penable && pready`), and pushes observed transactions into `mbx_mon_scb`. Intentionally corrupts `prdata` on the 2nd and 4th read (XOR with `0xFFFFFFFF`) to demonstrate error detection.
- **Scoreboard** — maintains a reference memory model (`ref_mem[256]`). On writes, updates the reference. On reads, compares DUT output against the reference and logs PASS/FAIL with running statistics.

---

## 📂 File Structure

```
.
├── apb_design.sv        # DUT: apb_slave module + apb_if interface
├── tb_apb_design.sv      # Testbench: transaction, generator, driver, monitor, scoreboard, environment, tb top
├── docs/
│   ├── tb_architecture.png   # Architecture diagram (add here)
│   └── waveform.png          # Simulation waveform (add here)
└── README.md
```

---

## ⚙️ DUT — APB Slave

A simple 256-word (32-bit) memory slave:
- `pready` is tied high (`1'b1`) — single-cycle access, no wait states.
- On `psel && penable`: writes `pwdata` to `mem[paddr[7:0]]` if `pwrite`, else registers `mem[paddr[7:0]]` into `prdata`.
- Synchronous active-low reset (`presetn`) clears memory and `prdata`.

---

## 🧪 Verification Plan

| # | Step | Description |
|---|------|-------------|
| 1 | Write Phase | 10 randomized writes to random addresses `[0:255]`, addresses saved by generator |
| 2 | Read Phase | 10 reads issued to the exact same 10 addresses used in the write phase |
| 3 | Checking | Scoreboard compares each read's `prdata` against the value written to that address |
| 4 | Fault Injection | Monitor corrupts `prdata` on reads #2 and #4 to verify the scoreboard actually catches mismatches (not just always passing) |

**Expected result:** 10/10 writes logged, **8 PASS / 2 FAIL** on reads (fails on reads #2 and #4 only).

---

## ▶️ Running the Simulation

```bash
xrun -Q -unbuffered -timescale 1ns/1ns -sysv -access +rw design.sv testbench.sv
```

Sample expected log tail:

```
[SCB] STATISTICS -> PASS: 8 | FAIL: 2
Simulation complete via $finish(1) at time 800 NS + 0
```

---

## 📊 Waveform

<img width="1007" height="251" alt="image" src="https://github.com/user-attachments/assets/d0c86994-5c21-46a9-b5e1-9ebf413d4341" />

Key signals to observe (via `.vif` scope in EPWave):
`paddr`, `pwrite`, `psel`, `penable`, `pwdata`, `prdata`, `pready`

---

## ✅ Status

Testbench is functional and verified — write/read data integrity confirmed, and intentional fault injection is correctly caught by the scoreboard.
