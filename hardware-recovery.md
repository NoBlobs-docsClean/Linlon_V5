<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Watchdog, Fault Detection, and Hardware Recovery (`hardware-recovery`)

This document specifies the hardware fault detection, diagnostic, and recovery mechanisms in the Linlon V5 VPU (`0x56635002`): internal watchdog timer (`0xFF02_5000`), per-bus-master MMU faults (`0xFF02_4000`), RASC CPU exceptions (`CR1`/`CR2`/`CR18`/`SR`), SDMA faults (`0xFF02_1F00`), the `DUMP` command, and the host-initiated forced termination protocol (`TERMINATE`).

---

## 1. Core Hardware Watchdog (`0xFF02_5000`) (`[P][D]`)

The core has a dedicated watchdog timer block at **`0xFF02_5000`**, configured according to the MVE option `MVE_SET_OPT_INDEX_WATCHDOG_TIMEOUT` (`42`, where `0` = disabled):

| Address | Offset | Width | Access | Function |
|---|---:|---:|---|---|
| **`0xFF02_5000`** | `+0x00` | 32b | W | **Watchdog Comparator / Deadline:** fires IRQ bit 4 (`0x10`) when counter `+0x10` reaches this value |
| **`0xFF02_5010`** | `+0x10` | 32b | R | **Free-Running Timer:** monotonic watchdog timebase |
| **`0xFF02_5014`** | `+0x14` | 32b | W | **Watchdog IRQ Clear:** writing **`1`** clears the pending interrupt |

### 1.1 Normative Watchdog Operation (`[D]`)

- **Arm / Rearm (Kick):**
  ```c
  uint32_t now = *(volatile uint32_t *)0xFF025010u;
  *(volatile uint32_t *)0xFF025000u = now + timeout_ticks;
  ```
  - Enable bit 4 (`0x10`) in the supervisor RASC's IRQ controller (`0xFF02_6000 + rasc_id * 0x1000`).
  - Rearm on each frame/stripe progress or when handling work events.
- **When the Watchdog IRQ fires (bit 4 = `0x10` in `IRQ_STATUS`):**
  1. Write **`1`** to `0xFF02_5014` to clear the hardware interrupt.
  2. Optionally record diagnostics (block states and RASC context) into `ram_print`.
  3. Send **`MVE_RESPONSE_CODE_ERROR`** (`2010`) with **`error_code = MVE_ERROR_WATCHDOG` (`9`)** and a descriptive message in `MSG_OUTQ`, followed by `CR14 = 0` and `0xFF02B028 = 1`.
  4. Enter a safe/halted state awaiting `TERMINATE` from the host.

> **Host-side watchdog (`[P][O]`):** Independently of the firmware's internal watchdog, the Linux driver maintains its own session watchdog (observed default: `30,000 ms`). If the firmware hangs without responding, the driver invokes `mvx_lsid_terminate` and reports an error to the V4L2 layer.

---

## 2. `MMU ABORT` Handling — 8-Bus-Master Controller (`0xFF02_4000`) (`[D]`)

When any internal unit attempts to access an unmapped virtual address (`AP = 0`) or violates page permissions (`RO`/`X`/`RW`), the hardware asserts **bit 3 (`0x0000_0008`)** in `IRQ_STATUS` of the supervisor RASC's interrupt controller.

The internal MMU block at **`0xFF02_4000`** maintains `FAULT` / `FADDR` register pairs for **8 bus masters**, with a stride of **`0x20` bytes** per master (`m = 0..7`):

```text
MMU_FAULT(m) = 0xFF02_4010 + m * 0x20    (32b R/W: != 0 indicates active fault; writing 0 clears)
MMU_FADDR(m) = 0xFF02_4014 + m * 0x20    (32b R:   32-bit virtual address that caused the fault)
```

| Master `m` | Bus Master Subsystem | `MMU_FAULT` (R/W0C) | `MMU_FADDR` (R) |
|---:|---|---|---|
| `0` | **RASC0** | `0xFF02_4010` | `0xFF02_4014` |
| `1` | **RASC1** | `0xFF02_4030` | `0xFF02_4034` |
| `2` | **RASC2** | `0xFF02_4050` | `0xFF02_4054` |
| `3` | **RASC3** | `0xFF02_4070` | `0xFF02_4074` |
| `4` | **RASC4** | `0xFF02_4090` | `0xFF02_4094` |
| `5` | **SDMA** | `0xFF02_40B0` | `0xFF02_40B4` |
| `6` | **VDMA** | `0xFF02_40D0` | `0xFF02_40D4` |
| `7` | **REF** | `0xFF02_40F0` | `0xFF02_40F4` |

### Mandatory `MMU ABORT` Handling Sequence

1. Iterate `m = 0 .. 7`:
   - Read `fault = *(volatile uint32_t *)(0xFF024010 + m * 0x20)`.
   - If `fault != 0`:
     - Read `faddr = *(volatile uint32_t *)(0xFF024014 + m * 0x20)`.
     - Log master `m`, `fault`, and `faddr` to `ram_print` / error message.
     - Clear the fault by writing **`0`** to `0xFF024010 + m * 0x20`.
2. Send **`MVE_RESPONSE_CODE_ERROR`** (`2010`) with **`error_code = MVE_ERROR_ABORT` (`1`)** to the host (`CR14 = 0` + `0xFF02B028 = 1`).
3. Cease video operations in the session (fatal error) and await `TERMINATE` from the host.

---

## 3. CPU Exceptions (`RASC ABORT`) (`[D]`)

When a synchronous exception hits entries `1..3` of the vector table in `CR0`:

1. The stub saves `r0..r14` on the stack (`push r0-r14`), reads:
   - **`CR2`** (`mfcr r, 2`) = `FAR` (*Fault Address Register*),
   - **`CR1`** (`mfcr r, 1`) = `PC` of the faulting instruction,
   - **`SR`** (`mfsr r`) = *Status Register*,
   - **`CR18`** (`mfcr r, 0x12`) = `RASC_ID` (`0..4`),
   and pushes `(FAR, PC, SR)` onto the top of the frame.
2. The handler logs `RASC_ID`, `PC`, `SR`, `FAR`, and `R[0..14]` (and optionally dumps the stack and all 8 MMU bus masters at `0xFF02_4010+`).
3. Sends `MVE_RESPONSE_CODE_ERROR` (`MVE_ERROR_ABORT = 1`) if the MVE channel is usable and enters a safe halt (never executes `rte` on a fatal abort).

---

## 4. DMA and Internal Block Faults (`[D]`)

- **SDMA fault:** Read `*(volatile uint32_t *)0xFF021F00` (SDMA controller fault register). A non-zero value indicates an SDMA channel transfer error; channel details can be inspected at `0xFF02_1000` (status), `0xFF02_1600` (pending devices), and `0xFF02_1804`/`0xFF02_180C` (descriptors).
- **Recoverable bitstream errors (non-fatal):** During decoding, corrupted headers or slices must **not** abort the session with `MVE_RESPONSE_CODE_ERROR`; instead:
  - Emit `MVE_RESPONSE_CODE_EVENT` (`2016`) with `MVE_EVENT_ERROR_STREAM_CORRUPT` (`1`) or `MVE_EVENT_ERROR_STREAM_NOT_SUPPORTED` (`2`).
  - Mark any affected frames with `MVE_BUFFER_FRAME_FLAG_CORRUPT` (`0x2000`) when returning them in `BUF_OUTRQ`.

---

## 5. `MVE_REQUEST_CODE_DUMP` Command (`1008`) (`[P][D]`)

When the host sends `MVE_REQUEST_CODE_DUMP`:
1. **Protected session:** If the session is protected, **forbid** the dump and report an error (`MVE_ERROR_UNSUPPORTED` / `MVE_ERROR_INVALID_STATE`) to avoid leaking protected content metadata.
2. **Normal session:** Write a diagnostic snapshot to `ram_print` (`0x1010_0000`), covering:
   - Internal silicon revision (`0xFF02_B06C`) and core clock (`0xFF02_B004`),
   - Logical session state and MVE queues,
   - Status registers of the video blocks (`BITSTREAM`, `REF`, `MC`, `TRF`, `DBL`, `SDMA`, `VDMA`).
3. Send **`MVE_RESPONSE_CODE_DUMP`** (`2012`, `size = 0`) to the host.

---

## 6. Host-Initiated Forced Recovery (`TERMINATE` and `RESET`) (`[P]`)

If a session fails to respond after a fatal error or timeout:
1. The host removes the LSID from `JOBQUEUE` (with `ENABLE = 0` → update `JOBQUEUE` → `ENABLE = 1`).
2. The host writes `1` to `LSID.TERMINATE` (`+0x18`) and polls until `TERMINATE == 0` (the hardware immediately aborts all cores executing that LSID and frees their entries in `CORELSID`).
3. The host unmaps the LSID (`SCHED = 0`, `ALLOC = 0`) and frees the MMU tables.
4. In the event of suspend/resume or a global block hang, the host executes `reset_hw()`: `RESET (0x50) = 1` → poll `CORELSID == corelsid_mask` (`0xFF` on K1, `0xFFFF` on K3) → `CLKFORCE (0x24) = 0`.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
