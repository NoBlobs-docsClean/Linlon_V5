<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Consolidated Internal Register and SRAM Map (`0xFF00_0000` – `0xFF2F_8FFF`)

In the 32-bit virtual address space of the MVE 3.x protocol, the range **`0xF000_0000` – `0xFFFF_FFFF`** is reserved for internal VPU use (`[P]`). The RASC CPUs access shared internal SRAMs, interrupt controllers, watchdog, internal MMU, DMAs, and the 6 video accelerator blocks through the range **`0xFF00_0000` – `0xFF2F_8FFF`** (`[O][D]`).

> **Note on address construction (`[O]`):** In the RASC ISA, `0xFFxxxxxx` addresses are constructed with `movn rd, imm`, whose actual semantics on silicon are **bitwise NOT (`~imm`)** — e.g., `movn r, 0xFD4FFF` produces `0xFF02_B000`. All addresses below reflect the real alignment validated on hardware.

---

## 1. General Map of the Core Internal Space (`[O][D]`)

| Base | Size / Range | Hardware Block | Description |
|---|---|---|---|
| **`0xFF00_0000`** | On-chip SRAM | **Session SRAM** (includes SDMA STP window at `0xFF00_1C00`) | SRAM shared across **all physical cores** of the same LSID (layout defined by firmware) |
| **`0xFF01_0000`** | `8 KiB` (`0xFF01_0000..1FFF`) | **Core-Local Shared SRAM** | SRAM shared among the **5 RASC CPUs of the same core** (free layout for intra-core synchronization) |
| **`0xFF02_1000`** | `4 KiB` | **SDMA** (*System DMA*) | System DMA channels, descriptors, fault register (`+0xF00`), and config (`+0xF04`) |
| **`0xFF02_2000`** | `4 KiB` | **VDMA** (*Video DMA*) | Video DMA controller (planar, AFBC, rotation, downscale) |
| **`0xFF02_3000`** | `4 KiB` | **RASC Control** | RASC1..4 reset and boot vectors, and inter-RASC doorbells |
| **`0xFF02_4000`** | `4 KiB` | **Core MMU Fault Controller** | MMU control (`+0x0C`) and 8 fault/address banks (`+0x10/+0x14`, stride `0x20`) for 8 bus masters |
| **`0xFF02_5000`** | `4 KiB` | **Watchdog / Core Timer** | Watchdog comparator (`+0x00`), free-running timer (`+0x10`), and IRQ clear (`+0x14`) |
| **`0xFF02_6000`–`0xFF02_A000`** | `5 × 4 KiB` | **5 × Per-RASC IRQ Controllers** | `0xFF02_6000 + rasc_id * 0x1000` (`RASC0..4`): `IRQ_STATUS`, `IRQ_ENABLE`, `IRQ_MASK2` |
| **`0xFF02_B000`** | `4 KiB` | **SYSCTRL / Core ID / Host IRQ / Spinlock** | `CORE_ID`, `CORE_CLK`, host-to/from IRQ, switch-out, hardware spinlock, inter-core IRQ, `HARDWARE_ID`, internal revision, and DSL capabilities |
| **`0xFF04_0000`** | `256 KiB` stride | **`BITSTREAM`** | Bitstream front-end, entropy (CABAC/CAVLC, VP8/VP9 boolean decoder), and motion vector prediction |
| **`0xFF08_0000`** | `256 KiB` stride | **`REF`** | Reference subsystem and AFBC cache |
| **`0xFF0C_0000`** | `256 KiB` stride | **`ME`** *(encode only)* | *Motion Estimation* |
| **`0xFF10_0000`** | `256 KiB` stride | **`MC`** | *Motion Compensation* |
| **`0xFF14_0000`** | `256 KiB` stride | **`TRF`** | Forward/inverse transform and quantization |
| **`0xFF18_0000`** | `256 KiB` stride | **`DBL`** | Deblocking filter and SAO |
| **`0xFF2F_8000`** | `4 KiB` | **Auxiliary System Block** | Auxiliary system control/status (`+0x00..+0x28`, 32-bit) |

> **Architectural pattern (`[D]`):** The 6 video accelerators are mapped with a **fixed stride of `0x0004_0000` (256 KiB)**: `BITSTREAM (0xFF04_0000)`, `REF (0xFF08_0000)`, `ME (0xFF0C_0000)`, `MC (0xFF10_0000)`, `TRF (0xFF14_0000)`, and `DBL (0xFF18_0000)`.

---

## 2. `SYSCTRL` / Host IRQ / Spinlock Block (`0xFF02_B000`) (`[O][D]`)

| Address | Offset | Width | Access | Provenance | Hardware Function |
|---|---:|---:|---|---|---|
| `0xFF02_B000` | `+0x00` | 32b | R | `[D]` | **`CORE_ID`:** Index of the current physical core (`0..1` on K1, `0..3` on K3) |
| `0xFF02_B004` | `+0x04` | 32b | R | `[D]` | **`CORE_CLK`:** Monotonic core clock counter (used in dumps and timestamps) |
| `0xFF02_B020` / `+0x24` | `+0x20`/`+0x24` | 32b | R | `[D]` | Auxiliary bus status/counter registers |
| **`0xFF02_B028`** | **`+0x28`** | **32b** | **W** | **`[O]`** | **Trigger IRQ → Host:** Writing **`1`** (after `CR14 = 0`) sets `IRQVE[lsid]` and triggers IRQ 74 to Linux |
| **`0xFF02_B02C`** | **`+0x2C`** | **32b** | **W** | **`[D]`** | **Clear `IRQHOST`:** Writing **`0`** clears the Host → Firmware interrupt (bit 18 of `IRQ_STATUS`) |
| **`0xFF02_B030`** | **`+0x30`** | **32b** | **W** | **`[D]`** | **Core Switch-Out / Release:** Writing **`1`** terminates core execution and releases its nibble in `CORELSID` |
| `0xFF02_B034` | `+0x34` | 32b | W | `[D]` | Core SYSCTRL configuration (receives **`0x5F`** at initialization) |
| **`0xFF02_B038`** | **`+0x38`** | **32b** | **R/W** | **`[D]`** | **Atomic Hardware Spinlock:** Write `((core_id + 1) << 4) \| rasc_id` only takes effect if currently `== 0`; readback until equal; writing **`0`** releases |
| **`0xFF02_B040 + c*4`** | `+0x40..+0x4C` | 32b | W | `[D]` | **Inter-Core Doorbell (core `c`):** Writing `1` raises IRQ bit 23 (`0x800000`) on core `c`; writing `0` clears |
| **`0xFF02_B068`** | **`+0x68`** | **32b** | **R** | **`[D]`** | **Internal `HARDWARE_ID`:** `0x56635002` on K1 and K3 (internal mirror of `0xC050_0000`) |
| **`0xFF02_B06C`** | **`+0x6C`** | **32b** | **R** | **`[D]`** | **Internal silicon revision:** internal hardware revision identifier |
| `0xFF02_B074` | `+0x74` | 32b | R | `[D]` | **Capabilities:** bit 29 (`0x2000_0000`) indicates extended DSL mode capability |
| `0xFF02_B07C` | `+0x7C` | 32b | R | `[D]` | **DSL Mode:** bit 0 (`0x1`) indicates DSL precision (`0` = standard/low, `1` = high) |

---

## 3. RASC Control (`0xFF02_3000`), MMU (`0xFF02_4000`), Watchdog (`0xFF02_5000`), and IRQ (`0xFF02_6000`–`0xFF02_A000`) Blocks (`[D]`)

| Block / Address | Width | Access | Function |
|---|---:|---|---|
| `0xFF02_3004` / `08` / `0C` / `10` | 32b | W | Reset of RASC1 / RASC2 / RASC3 / RASC4 (`1` = assert, `0` = release) |
| `0xFF02_3024` / `28` / `2C` / `30` | 32b | W | Entry address (boot PC) of RASC1 / RASC2 / RASC3 / RASC4 |
| `0xFF02_3040` / `44` / `48` / `4C` / `50` | 32b | W | Inter-RASC signal / doorbell channel A (`RASC0..4`) |
| `0xFF02_3060` / `64` / `68` / `6C` / `70` | 32b | W | Inter-RASC doorbell / ack channel B (`RASC0..4`, triggers/clears bit 1 of `IRQ_STATUS`) |
| `0xFF02_400C` | 32b | W | Core internal MMU control (initialized to **`0x0F`**) |
| `0xFF02_4010 + m*0x20` (`m=0..7`) | 32b | R/W | **MMU fault status for master `m`** (`0..4`=RASC0..4, `5`=SDMA, `6`=VDMA, `7`=REF; writing `0` clears) |
| `0xFF02_4014 + m*0x20` (`m=0..7`) | 32b | R | **MMU faulting virtual address for master `m`** (32-bit VA that caused the fault) |
| `0xFF02_5000` | 32b | W | **Watchdog Compare:** deadline = `[0xFF02_5010] + timeout` |
| `0xFF02_5010` | 32b | R | **Watchdog Free-Running Timer** |
| `0xFF02_5014` | 32b | W | **Watchdog IRQ Clear** (write `1`) |
| `0xFF02_6000 + r*0x1000` (`r=0..4`) | 32b | R | `IRQ_STATUS` of RASC `r` (b1=inter-RASC, b3=MMU, b4=Watchdog, b17=internal event, b18=`IRQHOST`, b23=inter-core) |
| `0xFF02_6004 + r*0x1000` (`r=0..4`) | 32b | R/W | `IRQ_ENABLE` of RASC `r` |
| `0xFF02_6008 + r*0x1000` (`r=0..4`) | 32b | R/W | `IRQ_MASK2` / interrupt mode of RASC `r` |

---

## 4. DMA Engines: `SDMA` (`0xFF02_1000`) and `VDMA` (`0xFF02_2000`) (`[D]`)

| Block | Address | Width | Access | Observed Name / Role |
|---|---|---:|---|---|
| **SDMA** | `0xFF02_1000 + ch*4` | 32b | R/W | SDMA channel status (40 pairs = 80 channels `0..79`; bits `[7:5]` indicate activity) |
| **SDMA** | `0xFF02_1600 + dev*4` | 32b | R | Pending status per SDMA device (`0..55`) |
| **SDMA** | `0xFF02_1804 + ch*0x10` | 32b | R/W | Descriptor length (`srclen`/`dstlen`) of channel `ch` |
| **SDMA** | `0xFF02_180C + ch*0x10` | 32b | R/W | Channel `ch` control/STP (`stpen` = bit 26, `stpid` = bits `[23:16]`, bit 29 = valid length flag) |
| **SDMA** | **`0xFF02_1F00`** | 32b | R | **SDMA Fault** — SDMA controller fault register (value `!= 0` indicates channel error) |
| **SDMA** | **`0xFF02_1F04`** | 32b | W | Global SDMA configuration (initialized to **`0xB6`**) |
| **SDMA** | `0xFF00_1C00` (`32 × 4 B`) | 32b | R/W | **SDMA STP Window** (32 32-bit entries in Session SRAM for *Stream/Stripe Point* synchronization) |
| **VDMA** | `0xFF02_2000`–`0xFF02_229F` | 32b | R/W | **VDMA** controller registers (`0xFF02_227C..0xFF02_229C` = VDMA status/diagnostic window; detailed stripe-programming offsets pending — gap G1) |

---

## 5. Video Accelerator Blocks (`0xFF04_0000` – `0xFF18_0000`) (`[D]`)

The 6 video accelerator blocks occupy 256 KiB windows from `0xFF04_0000` to `0xFF18_0000`. The table below lists only the MMIO bases and the already-characterized control/status registers; the full bit-level specification for per-codec frame/stripe dispatch remains open in gap **G1** ([observed-sequences-and-gaps.md](observed-sequences-and-gaps.md)):

| Subsystem | MMIO Base | Mapped Control / Status Registers (`Offset` → Hardware Function) |
|---|---|---|
| **`BITSTREAM`** | **`0xFF04_0000`** | `+0x130`: available bits in input buffer · `+0x208`: decode/entropy control · `+0x420`: motion vector prediction sub-block status · `+0x468`: CABAC sub-block status (H.264/HEVC) · `+0x514`: VP8/VP9 boolean decoder status · `+0x628`: auxiliary arithmetic engine status |
| **`REF`** | **`0xFF08_0000`** | `+0x00`: reference subsystem status · `+0x08`: configuration · `+0x44`: stop mask · `+0x48`: active references |
| **`ME`** | **`0xFF0C_0000`** | Motion estimation block (used in **encode**; internal offsets pending in **G1**) |
| **`MC`** | **`0xFF10_0000`** | `+0x20`: trigger (*start*) · `+0x28`: status |
| **`TRF`** | **`0xFF14_0000`** | `+0x08`: status · `+0x64`: diagnostic status |
| **`DBL`** | **`0xFF18_0000`** | `+0x00`: main control · `+0x08`: main status · `+0x50`: secondary control · `+0x54`, `+0x58`, `+0x5C`: additional status registers |

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
