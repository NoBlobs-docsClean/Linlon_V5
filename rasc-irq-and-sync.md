<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# RASC Interrupts, Doorbells, and Multi-Core Synchronization (`rasc-irq-and-sync`)

This document specifies the interrupt model of the 5 RASC CPUs per core, the vector table configured in `CR0`, the 5 dedicated interrupt controllers (`0xFF02_6000`–`0xFF02_A000`), delivery and clearing of `IRQHOST` (Host → Firmware), inter-RASC signaling (`0xFF02_3000`), inter-core doorbells (`0xFF02_B040+`), and the hardware spinlock (`0xFF02_B038`).

---

## 1. Exception and Interrupt Vector Table (`CR0` — VBAR) (`[D]`)

Each RASC CPU has a vector table base register in **`CR0`**. The pointer written to `CR0` (loaded via `lpc`) points to a contiguous array of **four 32-bit virtual addresses** (16 bytes):

| Index | Offset | Hardware Event | Expected Return |
|---:|---:|---|---|
| `0` | `+0x00` | **External IRQ** (signaled by controller `0xFF02_6000 + rasc_id * 0x1000`) | Save context (`push r0-r14`, `CR2`/`CR1`/`SR`), handle IRQ, restore, and execute **`rte`** |
| `1` | `+0x04` | **Exception / Abort 1** (e.g., instruction abort / illegal access) | Save context, report `RASC ABORT` (`PC`/`SR`/`FAR`/`R0..R14`), and halt |
| `2` | `+0x08` | **Exception / Abort 2** (e.g., data abort / MMU) | Save context, report `RASC ABORT`, and halt |
| `3` | `+0x0C` | **Exception / Abort 3** (e.g., reserved exception / fatal error) | Save context, report `RASC ABORT`, and halt |

> **Addressing note (`[D]`):** The 4 table entries are 32-bit virtual addresses (base `0x1000 + file_offset` in instance 0). The hardware dispatches indirectly through the corresponding entry when an IRQ or exception occurs.

---

## 2. Local CPU Interrupt Control (`clri` / `seti`) (`[P][D]`)

The `clri` and `seti` instructions take a scalar register (`clri rN` / `seti rN`):

| Instruction | Hardware Semantics |
|---|---|
| **`clri rN`** | Saves the previous interrupt-enable state into **`rN`** and **disables** interrupts on the current RASC CPU |
| **`seti rN`** | Restores the interrupt-enable state from **`rN`** (safe nestable critical section!) |

**Normative critical section pattern:**
```asm
clri r3          ; save previous state in r3 and mask IRQs
; ... critical section ...
seti r3          ; restore previous state from r3
```

---

## 3. Per-RASC Interrupt Controllers (`0xFF02_6000` – `0xFF02_A000`) (`[D]`)

Each of the 5 RASC CPUs inside a core has its own interrupt controller bank, spaced by **`0x1000` (4 KiB)**:

```text
IRQ_CTRL_BASE(rasc_id) = 0xFF02_6000 + rasc_id * 0x1000    (rasc_id = 0..4)
```

| RASC CPU | IRQ Controller MMIO Base |
|---:|---|
| **RASC0** | `0xFF02_6000` |
| **RASC1** | `0xFF02_7000` |
| **RASC2** | `0xFF02_8000` |
| **RASC3** | `0xFF02_9000` |
| **RASC4** | `0xFF02_A000` |

### 3.1 Registers of Each Bank (`+0x00`, `+0x04`, `+0x08`)

| Offset | Width | Access | Logical Name | Function |
|---:|---:|---|---|---|
| `+0x00` | 32b | R | `IRQ_STATUS` | Bitmask of pending interrupts for this RASC |
| `+0x04` | 32b | R/W | `IRQ_ENABLE` | Primary IRQ enable mask (bit `1` = enabled) |
| `+0x08` | 32b | R/W | `IRQ_MASK2` / `IRQ_MODE` | Secondary mask / mode configuration for IRQ lines |

### 3.2 Bit Map of `IRQ_STATUS` (`+0x00`) and `IRQ_ENABLE` (`+0x04`)

The confirmed bits in the RASC CPU interrupt controller are:

| Bit | Hex Mask | Hardware Source | How to Clear / Acknowledge Source |
|---:|---|---|---|
| `1` | `0x0000_0002` | **Inter-RASC Doorbell** (signal sent by another RASC CPU in the same core) | Write **`0`** to `0xFF02_3040 + rasc_id * 4` and/or `0xFF02_3060 + rasc_id * 4` |
| `3` | `0x0000_0008` | **`MMU ABORT`** (translation/permission fault in any of the 8 bus masters) | Read `0xFF02_4010 + m * 0x20` (`m = 0..7`), read the faulting address at `+0x14`, and write **`0`** to `+0x10` (see [hardware-recovery.md](hardware-recovery.md)) |
| `4` | `0x0000_0010` | **`WATCHDOG` Timer IRQ** (`0xFF02_5000` comparator reached) | Write **`1`** to **`0xFF02_5014`** |
| `17` | `0x0002_0000` | **Internal Job / Subsystem Event** | Handled according to job queue state / internal signaling |
| **`18`** | **`0x0004_0000`** | **`IRQHOST` (Host → Firmware)** — triggered when the Linux driver writes `1` to `LSID.IRQHOST` | Write **`0`** to **`0xFF02_B02C`** |
| **`23`** | **`0x0080_0000`** | **Inter-Core Doorbell** — triggered when another physical core writes `1` to `0xFF02_B040 + core_id * 4` | Write **`0`** to **`0xFF02_B040 + core_id * 4`** (where `core_id = [0xFF02_B000]`) |

> **Per-RASC mask configuration (`[D]`):** Which IRQ lines to enable in `IRQ_ENABLE` (`+0x04`) and `IRQ_MASK2` (`+0x08`) on each RASC CPU depends solely on the architecture chosen by the clean-room firmware. For example, the RASC designated as the MVE supervisor enables the bits corresponding to `MMU ABORT` (bit 3), `WATCHDOG` (bit 4), `IRQHOST` (bit 18), and Inter-Core (bit 23), while auxiliary CPUs enable the Inter-RASC doorbell (bit 1) and the lines of the accelerators they manage.

---

## 4. Host ↔ Firmware Signaling (`0xFF02_B028` and `0xFF02_B02C`) (`[O][D]`)

| Direction | MMIO Register | Operation | Effect |
|---|---|---|---|
| **Firmware → Host** | **`0xFF02_B028`** (`[O]`) | `CR14 = 0`, then write **`1`** | Sets the LSID bit in `IRQVE` (`0xC050_0018`) and triggers IRQ 74 to Linux |
| **Host → Firmware (receive)** | **`0xFF02_6000 + rasc_id * 0x1000`** (bank of the RASC servicing the host) (`[D]`) | Test **bit 18 (`0x0004_0000`)** of `IRQ_STATUS` | Indicates that the driver wrote `1` to `LSID.IRQHOST` (new messages/buffers or RPC response) |
| **Host → Firmware (clear)** | **`0xFF02_B02C`** (`[D]`) | Write **`0`** | Clears the pending `IRQHOST` in the core hardware |

---

## 5. Inter-RASC (`0xFF02_3040..0xFF02_3070`) and Inter-Core (`0xFF02_B040+`) Doorbells (`[D]`)

### 5.1 Inter-RASC (Within the Same Core — Block `0xFF02_3000`)

| Channel | RASC0 | RASC1 | RASC2 | RASC3 | RASC4 | Usage |
|---|---|---|---|---|---|---|
| **Doorbell / Signal A** | `0xFF02_3040` | `0xFF02_3044` | `0xFF02_3048` | `0xFF02_304C` | `0xFF02_3050` | Write `1` to signal (or `0` to clear) the target RASC |
| **Doorbell / Signal B** | `0xFF02_3060` | `0xFF02_3064` | `0xFF02_3068` | `0xFF02_306C` | `0xFF02_3070` | Wakes the target RASC (bit 1 in `IRQ_STATUS`); write `0` in the handler to clear |

### 5.2 Inter-Core (Between Physical Cores of the Same LSID — Block `0xFF02_B000`)

- **Current core ID:** `core_id = *(volatile uint32_t *)0xFF02B000` (`0..1` on K1; `0..3` on K3).
- **Current RASC ID:** `rasc_id = mfcr(0x12)` (`0..4`).
- **Send IRQ to `target_core`:**
  1. Execute **`CR14 = 0`** to ensure visibility of Session SRAM (`0xFF00_0000`) and `master_rw`.
  2. Write **`1`** to **`*(volatile uint32_t *)(0xFF02B040 + target_core * 4)`**.
- **Clear received inter-core IRQ (bit 23 = `0x0080_0000` in `IRQ_STATUS`):**
  - Write **`0`** to **`*(volatile uint32_t *)(0xFF02B040 + my_core_id * 4)`**.

---

## 6. Hardware Spinlock / Mutex (`0xFF02_B038`) (`[D]`)

Register **`0xFF02_B038`** implements an **atomic hardware mutex** shared by all physical cores and all RASC CPUs of a session:

- **Hardware rule:** A non-zero write to `0xFF02_B038` is only accepted if the register is currently **`0`** (free). If already held by another owner, the write is ignored and the subsequent read returns the current owner's token. Writing **`0`** releases the lock.
- **Unique token format per `(core_id, rasc_id)`:**
  ```c
  uint32_t core_id = *(volatile uint32_t *)0xFF02B000u;
  uint32_t rasc_id = rasc_mfcr_0x12(); /* CR18: 0..4 */
  uint32_t token   = ((core_id + 1u) << 4) | (rasc_id & 0xFu);
  /* Note: core_id + 1 guarantees token is ALWAYS != 0! */
  ```
- **Acquisition sequence (reentrant-safe):**
  1. If `*(volatile uint32_t *)0xFF02B038 == token`, this CPU already holds the lock (return 0 / already acquired).
  2. Disable local interrupts (`clri saved`).
  3. In a loop:
     - Write `token` to `0xFF02B038`.
     - Read `0xFF02B038` back.
     - Repeat while the read value `!= token`.
  4. Restore interrupts (`seti saved`) or keep them disabled during the critical section.
- **Release sequence:**
  1. Execute **`CR14 = 0`** to guarantee that all writes in the critical section are visible in Session SRAM / DRAM before releasing the lock.
  2. Write **`0`** to **`*(volatile uint32_t *)0xFF02B038`**.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
