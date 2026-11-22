<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Execution Environment — 5 RASC CPUs per Core

This document specifies the embedded execution environment inside each Linlon V5 VPU core: the organization of the 5 RASC CPUs (Master + 4 Workers), the complete catalog of control registers (`CR0`–`CR255`), instance management via `CR4`, counters and identification via `CR16`/`CR18`, and exception/stack handling.

> For the step-by-step hardware boot sequence, see [boot-core-init.md](boot-core-init.md). For the interrupt controller, doorbells, and hardware spinlock, see [rasc-irq-and-sync.md](rasc-irq-and-sync.md).

---

## 1. Multi-RASC Model per Core (`[P][D]`)

Each physical VPU core contains **5 independent RASC processors** sharing the Core-Local SRAM (`0xFF01_0000`, 8 KiB) and accessing the Session SRAM (`0xFF00_0000`, shared across all cores in the LSID):

| RASC CPU | `RASC_ID` (`CR18`) | IRQ Controller | Architectural Role |
|---|---:|---|---|
| **RASC0 (Master)** | `0` | `0xFF02_6000` | The only CPU that starts automatically on hardware reset at `core_base + 0x1000`; programs worker vectors/resets at `0xFF02_3000` and initializes the core |
| **RASC1 (Worker 1)** | `1` | `0xFF02_7000` | Symmetric auxiliary processor (role defined by firmware; released via `0xFF02_3004`/`0xFF02_3024`) |
| **RASC2 (Worker 2)** | `2` | `0xFF02_8000` | Symmetric auxiliary processor (role defined by firmware; released via `0xFF02_3008`/`0xFF02_3028`) |
| **RASC3 (Worker 3)** | `3` | `0xFF02_9000` | Symmetric auxiliary processor (role defined by firmware; released via `0xFF02_300C`/`0xFF02_302C`) |
| **RASC4 (Worker 4)** | `4` | `0xFF02_A000` | Symmetric auxiliary processor (role defined by firmware; released via `0xFF02_3010`/`0xFF02_3030`) |

- **Flexibility rule for the clean-room team (`[O]`):** Except for the fact that only **RASC0** boots automatically on reset and each CPU has its own dedicated IRQ controller (`0xFF02_6000 + rasc_id * 0x1000`), the 4 workers (`RASC1..4`) are symmetric auxiliary processors whose division of responsibilities (MVE protocol, `IRQHOST`, watchdog, parsing, or video pipeline) is freely chosen by firmware software. In M1/M2, RASC0 alone can handle the entire MVE 3.5 protocol.

---

## 2. Control Register Catalog (`CR0` – `CR255`) (`[O][D]`)

The `mtcr rs, imm` and `mfcr rd, imm` instructions address a space of **256 control registers** (**8-bit** index, `0x00`–`0xFF`, proven on hardware `[O]`). The CR space is split between **CPU/system CRs (`0x00`–`0x1F`)** and **extended coprocessor/accelerator banks (`0x20`–`0xFF`)**:

### 2.1 RASC CPU Control Registers (`CR 0x00` – `0x1F`)

| CR (Dec / Hex) | Access | Provenance | Hardware-Confirmed Function |
|---|---:|---|---|
| **`CR0`** (`0x00`) | W | `[D]` | **VBAR (*Vector Base Address Register*):** Pointer to table of **4 32-bit vectors** (`[IRQ, Abort1, Abort2, Abort3]`) |
| **`CR1`** (`0x01`) | R | `[D]` | **EPC (*Exception PC*):** Address of the instruction that triggered the exception/abort |
| **`CR2`** (`0x02`) | R | `[D]` | **FAR (*Fault Address Register*):** Faulting virtual address on memory exceptions |
| **`CR3`** (`0x03`) | W | `[D]` | **Base CPU Configuration:** Initialized to **`0x0000_01A7`** at boot on each RASC |
| **`CR4`** (`0x04`) | R/W | `[D]` | **Firmware Instance Base (`slot << 24`):** Selects which 16 MiB window (`0x0000_0000`, `0x0100_0000`, `0x0200_0000`, `0x0300_0000`) maps this core instance |
| **`CR6`** (`0x06`) | W | `[D]` | **CPU Mode / Power Control:** Receives **`0x0000_0200`** in active boot mode; receives **`0x0000_0001`** before `wait` in final idle |
| **`CR11`** (`0x0B`) | W | `[D]` | **Auxiliary State/Reset Control:** Zeroed (`0`) after init and before switch-out |
| `CR12` / `CR13` (`0x0C`/`0x0D`) | W | `[D]` | Auxiliary CPU control registers (sporadic use; details open) |
| **`CR14`** (`0x0E`) | W | **`[O][D]`** | **Memory Barrier / Write-Buffer Drain:** Writing **`0`** guarantees global visibility of DRAM/SRAM writes before releasing spinlocks (`0xFF02_B038`), ringing doorbells, or raising host IRQ (`0xFF02_B028`). **Validated on K1 silicon.** |
| **`CR15`** (`0x0F`) | W | `[D]` | **Cache Maintenance (128 B lines):** Writing a `0x80`-aligned address performs a prefetch/invalidation operation on the corresponding cache line |
| **`CR16`** (`0x10`) | R | `[D]` | **Cycle Counter:** Free-running RASC CPU cycle counter (used for latency measurement and profiling) |
| **`CR18`** (`0x12`) | R | `[D]` | **`RASC_ID`:** Immutable identifier of the RASC CPU within the core (**`0`, `1`, `2`, `3`, or `4`**) |

### 2.2 Extended Coprocessor / Accelerator Banks (`CR 0x20` – `0xFF`) (`[O][D]`)

Beyond the CPU registers (`0x00`–`0x1F`), `mtcr`/`mfcr` access register banks of internal coprocessors/accelerators coupled to the RASC CPU pipeline (no public documentation; detailed per-register semantics remain open in gap **G2** in [observed-sequences-and-gaps.md](observed-sequences-and-gaps.md)):

- **Empirical silicon finding (`[O]` — K1):** Direct tests on SpacemiT K1 CPU RASC0 proved that writes to **`CR 0x90`, `CR 0x93`, and `CR 0x94`** have cache/state side effects on memory when critical data resides in unsynchronized RAM, and that `CR 0x94` and `CR 4` are decoded as distinct registers by the hardware.
- **Critical warning for the clean-room team (`[O]`):** Never mask the CR number to 4 bits (`& 0xF`), because all 8 bits are decoded by silicon. Avoid touching uncharacterized extended CRs (`>= 0x20`) during scalar bring-up; for Milestones **M1–M3**, only the CPU CRs (`CR0`, `CR1`, `CR2`, `CR3`, `CR4`, `CR6`, `CR11`, `CR14`, `CR15`, `CR16`, `CR18`) are required.

---

## 3. Status Register (`SR`), `clri`/`seti`, and Exception Handling (`[P][D]`)

### 3.1 Hardware Contract — Status Instructions and Exception Registers

- `mfsr rd` / `mtsr rs`: read and write the *Status Register* (`SR`).
- `clri rN`: **saves** the previous interrupt state into `rN` and **disables** interrupts.
- `seti rN`: **restores** the interrupt state from `rN`.
- `wait`: halts the RASC CPU until an enabled interrupt arrives.
- `rte`: returns from an exception/interrupt (used at the end of vector `0` of `CR0`).
- Upon entering any of the 4 vectors of `CR0`, the hardware exposes the faulting address in **`CR2` (`FAR`)**, the instruction address in **`CR1` (`PC`)**, and the status in **`SR` (`mfsr`)**. The `push r0-r14` instruction stores `r0` at the lowest address through `r14` (`lr`) at the highest address.

### 3.2 Illustrative Stack Exception Frame (Software Convention)

The exact layout saved on the stack by the IRQ/exception stub is a free choice for the clean-room implementation. As an illustrative pattern compatible with `push r0-r14`:
1. `push r0-r14` (15 words: `r0` at lowest address, `r14` at highest).
2. Read `CR2` (`FAR`), `CR1` (`PC`), and `SR` (`mfsr`) into scratch registers (e.g., `r12..r14`) and push them (`push r12-r14`, 3 words).
3. Pass `sp` in `r0` to the C handler:

| Offset from `sp` | Size | Field (illustrative) |
|---:|---:|---|
| `+0x00` | 4 B | `FAR` (`CR2`) |
| `+0x04` | 4 B | `PC` (`CR1`) |
| `+0x08` | 4 B | `SR` (`mfsr`) |
| `+0x0C` .. `+0x44` | 60 B | `R[0]` .. `R[14]` in ascending order |

---

## 4. Stack Rules and Multi-Instance Isolation (`[O][D]`)

1. Each of the 5 RASC CPUs **must** have an exclusive, non-overlapping stack inside the core instance's private BSS (pages with bit `1` in `bss_bitmap` outside `master_rw`).
2. Every `sp` initialization **must** use `lpc sp, stack_top_rascN` (PC-relative addressing), so it works identically across instances `0x0000_0000`, `0x0100_0000`, `0x0200_0000`, and `0x0300_0000`.
3. Page `core_base + 0x0000` (`0x0000..0x0FFF`) is unmapped (guard page); any page with bit `0` in `bss_bitmap` also triggers `MMU ABORT` on *stack overflow*.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
