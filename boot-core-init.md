<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Core Boot Sequence and Hardware Initialization (`boot-core-init`)

This document specifies the internal MMIO register and control register (`CR`) programming sequence required by each physical core of the **Arm China Linlon V5** VPU (`HARDWARE_ID = 0x56635002`) on SpacemiT K1 (2 cores) and K3 (4 cores) on every initial boot (*cold boot*) and resumption after *switch-out* (*warm boot*).

> **Provenance:** Internal MMIO offsets, initialization values, and handshake ordering are `[D]` (specification derived from silicon interaction), with RASC0 boot, `CR14`, `0xFF02B028`, and PC-relative stacks empirically confirmed on a real SpacemiT K1 board (`[O]`).

---

## 1. Overview of Initialization Phases (`[O][D]`)

When the VPU hardware scheduler assigns an LSID to one or more physical cores (`JOBQUEUE` + `SCHED = 1`), each physical core begins execution on **RASC0** at virtual address `core_base + 0x1000` (offset `0x00` of the `.fwb` file):

```text
1. Reset vector at offset 0x00 (`br _start`) jumps over struct mvx_fw_header (180 B)
2. RASC0: configures CR3 = 0x1A7
3. RASC0: publishes RASC worker entry addresses at 0xFF023024..0xFF023030
          and pulses reset (1 → 0) at 0xFF023004..0xFF023010
4. RASC0: executes memory barrier CR14 = 0 and acquires hardware spinlock (0xFF02B038)
5. RASC0: allocates instance slot (0..NCORES-1), programs its own CR4 = (slot << 24),
          and communicates the chosen slot to workers (e.g., via Core-Local SRAM at 0xFF010000)
6. RASC1..4: come out of reset, configure CR3 = 0x1A7, program CR4 = (slot << 24),
             and optionally perform cache maintenance/prefetch via CR15 (128 B lines)
7. Each RASC: configures CR6 = 0x200, PC-relative stack (`lpc sp, ...`), VBAR in CR0 (4-vector table),
              and interrupt mask in its dedicated controller (0xFF026000 + rasc_id * 0x1000)
8. Core hardware defaults initialization (MMU 0xFF02400C, SYSCTRL 0xFF02B034, SDMA 0xFF021F04)
9. Validation of HARDWARE_ID (0xFF02B068) and DSL capabilities (0xFF02B074 / 0xFF02B07C)
10. Publication of SWITCHED_IN in MSG_OUTQ + CR14 = 0 + host IRQ (0xFF02B028 = 1)
```

> **Note for Milestone M1/M2 (Minimal bring-up on RASC0):** If the initial clean-room firmware runs solely on **RASC0** (without waking RASC1–RASC4), steps 3, 5, and 6 may be omitted; it suffices to execute the `br _start` trampoline, initialize `sp` via `lpc`, configure `CR3 = 0x1A7` and `CR0` (VBAR), and operate directly on the MVE queues (`0x1007_9000`–`0x1007_F000`) + `CR14 = 0` + `0xFF02B028 = 1` (`[O]`).

---

## 2. Phase A — RASC0 Startup and Releasing RASC Workers (`[D]`)

### 2.1 Initial CPU Setup and Worker Publication

1. **Base CPU configuration:** Write **`0x0000_01A7`** to **`CR3`** (`mtcr r, 3`).
2. **RASC worker entry vectors (block `0xFF02_3000`):**
   - Compute each worker's entry address via `lpc` (PC-relative).
   - Write the entry addresses to the reset vector registers:

   | Register | Width | Function |
   |---|---:|---|
   | `0xFF02_3024` | 32b W | Entry (boot) address of **RASC1** |
   | `0xFF02_3028` | 32b W | Entry (boot) address of **RASC2** |
   | `0xFF02_302C` | 32b W | Entry (boot) address of **RASC3** |
   | `0xFF02_3030` | 32b W | Entry (boot) address of **RASC4** |

3. **RASC worker reset pulse (block `0xFF02_3000`):**
   - For each worker to activate, write **`1`** and then **`0`** to its respective reset register (`1` = assert reset, `0` = release/run):

   | Register | Width | Function |
   |---|---:|---|
   | `0xFF02_3004` | 32b W | Reset of **RASC1** (`1` → `0`) |
   | `0xFF02_3008` | 32b W | Reset of **RASC2** (`1` → `0`) |
   | `0xFF02_300C` | 32b W | Reset of **RASC3** (`1` → `0`) |
   | `0xFF02_3010` | 32b W | Reset of **RASC4** (`1` → `0`) |

   *(Note: RASC1 and RASC2 may be released right at the beginning by RASC0; RASC3 and RASC4 may be released afterward by RASC0 or by an already active worker before starting the video pipeline.)*

4. **Memory barrier:** Write **`0`** to **`CR14`** (`mtcr r, 0x0E`).

---

## 3. Phase B — Instance Slot Election and `CR4` Programming (`[D]`)

In multi-core sessions (`NCORES = 2` on K1, `4` on K3), each physical core entering execution must obtain a **unique instance index** (`slot` in `0 .. ncores - 1`) to map its private BSS (`slot * 0x0100_0000`) without colliding with other cores:

1. **Acquire the hardware spinlock (`0xFF02_B038`):**
   - Read `core_id = *(volatile uint32_t *)0xFF02B000` (`0 .. NCORES - 1`).
   - Form any unique non-zero token per requester (hardware rule: `0xFF02_B038` accepts any value `!= 0` when free at `0`; a simple convention is `token = ((core_id + 1) << 4) | rasc_id`).
   - Write `token` to `0xFF02B038` and perform a **mandatory poll** reading `0xFF02B038` until reading back exactly `token` (see [rasc-irq-and-sync.md](rasc-irq-and-sync.md)).
2. **Allocate a free slot in Session SRAM (`0xFF00_0000`):**
   - Since `0xFF00_0000` is SRAM shared across all cores of the LSID, the firmware picks a free slot `slot` (`0 .. ncores - 1`) and marks it occupied under spinlock protection.
   - If no slot is temporarily available, release the spinlock (`CR14 = 0`, `0xFF02B038 = 0`), wait a short delay, and retry.
3. **Configure the instance base register (`CR4`) — hardware contract:**
   - Write **`slot << 24`** (`0x0000_0000`, `0x0100_0000`, `0x0200_0000`, `0x0300_0000`, ...) to **`CR4`** (`mtcr r, 4`).
   - This register informs the RASC CPU addressing logic which 16 MiB instance window (`MVE_MEM_REGION_FW_INSTANCE<slot>`) belongs to this core.
4. **Propagate the slot to this core's RASC workers via Core-Local SRAM (`0xFF01_0000`):**
   - **Hardware contract:** The window `0xFF01_0000`–`0xFF01_1FFF` (8 KiB) is SRAM shared exclusively among the 5 RASC CPUs of the same core; the internal layout is entirely defined by firmware software.
   - **Illustrative synchronization convention:** RASC0 may publish `(slot << 24) | 1` (bit 0 as a ready flag, allowing `slot == 0` to be distinguished from initial zero) in a chosen word of Core-Local SRAM (e.g., `0xFF01_0000`), execute **`CR14 = 0`** for local visibility, and then release the spinlock by writing **`0`** to `0xFF02B038`.

---

## 4. Phase C — Startup Sequence of Each RASC Worker (`RASC1..4`) (`[D]`)

Each worker RASC CPU executes the following steps upon coming out of reset:

1. **CPU configuration (hardware contract):** Write **`0x0000_01A7`** to **`CR3`**.
2. **Slot synchronization and optional cache maintenance (`CR15`):**
   - **Hardware contract:** Each worker must program its own **`CR4 = slot << 24`** with the same `slot` allocated to the core, and register **`CR15`** accepts **128-byte**-aligned addresses (`addr & ~0x7F`, stride `0x80`) for cache line maintenance/prefetch.
   - **Illustrative wait example:** While waiting for RASC0's signal in Core-Local SRAM (e.g., `v = *(volatile uint32_t *)0xFF010000`, waiting for `v != 0` under the `(slot << 24) | 1` convention), the worker may walk `[start, end)` of its code (computed via `lpc`, aligned to `0x80`) writing `addr` to `CR15`, and then program **`CR4 = v & ~1`**.
3. **Configure control/stack (hardware contract):**
   - Write **`0x0000_0200`** to **`CR6`**.
   - Load this worker's exclusive stack via **`lpc sp, stack_top_rascN`**.
4. **Configure vector table (`CR0`, hardware contract):**
   - Load via `lpc` the address of an aligned table of four 32-bit pointers (`[irq_handler, abort0, abort1, abort2]`) and write it to **`CR0`** (`mtcr r, 0`).
5. **Call the worker's main function** (`bsr worker_main`). Upon return/exit, execute `wait; br .`.

*(On RASC0, after configuring `CR4`, it likewise sets `CR6 = 0x200`, `lpc sp, stack_top_rasc0`, copies any `.data` from ROM/text to BSS on first load, zeroes **`CR11 = 0`**, configures **`CR0`** with its vector table, and proceeds to block initialization.)*

---

## 5. Phase D — Core Block Initialization and Silicon Validation (`[D]`)

Before accepting video processing commands or host interrupts, the core must program the default configuration registers for `0x56635002` silicon:

### 5.1 Core Hardware Initialization Values

| MMIO Register | Width | Mandatory Value | Hardware Role |
|---|---:|---:|---|
| **`0xFF02_400C`** | 32b W | **`0x0000_000F`** | Configuration / enable of the core's internal MMU control block |
| **`0xFF02_B034`** | 32b W | **`0x0000_005F`** | Initial core SYSCTRL configuration |
| **`0xFF02_1F04`** | 32b W | **`0x0000_00B6`** | Initial global configuration of the **SDMA** controller |

### 5.2 Hardware Version and DSL Mode Validation (`0xFF02_B068`..`0xFF02_B07C`)

The firmware can verify at runtime which silicon it is running on directly through the internal SYSCTRL block (`0xFF02_B000`):

| MMIO Register | Width | Access | Function |
|---|---:|---|---|
| **`0xFF02_B068`** | 32b | R | **Internal `HARDWARE_ID`:** returns the same value as host register `0xC050_0000` (`0x56635002` on K1 and K3). To check for the Linlon V5 family, compare `(val >> 16) == 0x5663` (or `(val & 0xFFFF0000) == 0x56630000`), masking out revision and patch. |
| **`0xFF02_B06C`** | 32b | R | **Internal hardware revision:** internal silicon revision identifier (useful in diagnostic dumps). |
| **`0xFF02_B074`** | 32b | R | **Hardware capabilities:** bit 29 (`0x2000_0000`) indicates support for extended-precision flexible downscale (DSL) mode. |
| **`0xFF02_B07C`** | 32b | R | **Active DSL mode:** bit 0 (`0x0000_0001`) indicates whether the hardware reports high-precision DSL mode (`1`) or standard precision (`0`). |

### 5.3 Enabling Per-RASC Interrupt Controllers

Each RASC enables its desired interrupt sources in its bank `0xFF02_6000 + rasc_id * 0x1000` (full details in [rasc-irq-and-sync.md](rasc-irq-and-sync.md)) and executes `seti` to enable CPU interrupts.

---

## 6. Phase E — Core Shutdown / Switch-Out Sequence (`0xFF02_B030`) (`[D]`)

When the core must exit execution (in response to `MVE_REQUEST_CODE_SWITCH` after draining work and sending `MVE_RESPONSE_CODE_SWITCHED_OUT` on the last active core):

1. Disable local interrupts (`clri`) and clear the RASC interrupt controller as appropriate.
2. Zero `CR11 = 0` and execute barrier **`CR14 = 0`**.
3. If this is the last core in the session to exit, ensure `MVE_RESPONSE_CODE_SWITCHED_OUT` was written to `MSG_OUTQ`, followed by `CR14 = 0` and `0xFF02B028 = 1` (host IRQ).
4. Release the core's instance slot in Session SRAM (`0xFF00_0000`) and execute **`CR14 = 0`**.
5. **Trigger core release in the hardware scheduler:**
   - Write **`1`** to **`*(volatile uint32_t *)0xFF02B030`**.
   - Enter an infinite/halt loop (`br .` or `wait; br .`).
   - The hardware removes the LSID from the corresponding core in `CORELSID` (nibble returns to `0xF`), making the core available for another LSID or for `runtime_suspend`.

*(Note regarding RASC0 `wait` mode upon entering idle: RASC0 writes **`CR6 = 1`** before entering `wait; br .` when finishing its initialization routine.)*

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
