<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Clean-Room Specification — SpacemiT K1 / K3 VPU Firmware (Arm China Linlon V5)

This folder contains the behavioral, hardware architecture, ISA, and ABI contract specification for a **clean-room reimplementation** of the **Arm China Linlon V5** VPU firmware (`HARDWARE_ID = 0x56635002`, driver family `MVE_v52_v76`) integrated into the **SpacemiT Key Stone K1** (2 cores at 819.2 MHz) and **SpacemiT K3** (4 cores at 1 GHz) SoCs.

The primary target of this specification is the open **MVE 3.5** contract implemented by the GPL-2.0 Linux `mvx` driver (`compatible = "arm china,linlon-v5"`, firmware directory `linlon-v52_v76-80-2/`), executing on the VPU's **5 32-bit RASC CPUs per core**.

---

## Reading Rules and Provenance

Before starting any implementation, read [clean-room-process.md](clean-room-process.md).

Every technical statement in this specification follows three provenance levels:

- **`[P]` Public** — Normative contract derived directly from open sources: GPL-2.0 headers of the Linux `mvx` driver (`mve_protocol_def.h`, `mvx_hwreg.h`, `mvx_mmu.h`, `mvx_lsid.h`, `mvx_firmware_cache.h`), SpacemiT K1/K3 Device Trees, public SpacemiT datasheets/manuals, public Arm China technical presentations, and the public `ARM-software/rasc-devkit` toolchain (GPLv2, binutils 2.18 + GCC 4.2.3).
- **`[O]` Observed on Hardware** — Behavior, register value, MVE trace, or instruction semantics empirically verified on real boards (**SpacemiT K1 BPI-F3** and/or **SpacemiT K3 Pico ITX**), including execution of custom RASC code compiled with `rasc-elf-gcc` on K1 silicon.
- **`[D]` Derived Specification** — Behavioral requirement, internal MMIO register, control register (`CR`), or programming sequence required for `0x56635002` silicon to respond properly to the `[P]` contract and reproduce the `[O]` behavior, described exclusively in terms of hardware and public ABI.

---

## Documentation Map

### 1. Governance, Architecture, and Platform

| # | Document | Contents |
|---:|---|---|
| 01 | [clean-room-process.md](clean-room-process.md) | Clean-room firewall, what is allowed and what is forbidden, source hierarchy, clarification protocol, and acceptance milestones (M1–M6) |
| 02 | [overview.md](overview.md) | Linlon V5 VPU architecture (`0x56635002`), K1 (2 cores) vs. K3 (4 cores) comparison, 5-RASC-CPU-per-core model, and Host vs. Firmware boundary |
| 03 | [open-vs-closed.md](open-vs-closed.md) | Open sources (`mvx` GPL-2.0, `rasc-devkit` GPLv2) vs. proprietary blobs, MVE v2 vs. MVE 3.5, and K1 vs. K3 continuity |
| 04 | [soc-integration.md](soc-integration.md) | Integration in K1 and K3 SoCs: Device Tree, `0xC050_0000` MMIO window, IRQ 74, clock/reset/power, DVFS, hardware reset, and DMA translation (`phys_cpu2vpu`/`phys_vpu2cpu`) |

### 2. Host Registers, Scheduler, and MMU

| # | Document | Contents |
|---:|---|---|
| 05 | [host-registers.md](host-registers.md) | Complete map of APB registers at `0xC050_0000`: global (`0x00`–`0x50`), per-LSID (`0x0200 + 0x40 * lsid`), bitfields, fuses, and idle snapshots on K1/K3 |
| 06 | [lsid-and-scheduler.md](lsid-and-scheduler.md) | LSID contexts (4 LSIDs), transparent 2/4-core scalability, Map/Unmap/Terminate/PCB, `JOBQUEUE` (4 slots), Host↔VPU IRQ, and `SWITCH` |
| 07 | [mmu-and-memory-model.md](mmu-and-memory-model.md) | 2-level MMU (32-bit VA → 40-bit PA, 4 KiB pages, 32-bit PTE `AT`/`PA`/`AP`, `FLUSH_ALL`), complete MVE 3.x VA map, and coherency |

### 3. Firmware Container, RASC CPU, and Toolchain

| # | Document | Contents |
|---:|---|---|
| 08 | [firmware-container.md](firmware-container.md) | `.fwb` container: `struct mvx_fw_header` (180 B), load algorithm (`VA 0x1000`), `text_length` including header, `bss_bitmap` vs. `master_rw`, trampoline, and filenames |
| 09 | [rasc-isa.md](rasc-isa.md) | Complete RASC ISA reference (1–6 B + 1–4 B suffixes, `r0`–`r15`/`v0`–`v15`, **BCC3** instruction-count semantics, and silicon-validated corrections) |
| 10 | [rasc-execution-environment.md](rasc-execution-environment.md) | Environment of the 5 RASC CPUs: `CR0` (VBAR), `CR1` (PC), `CR2` (FAR), `CR4` (instance base), `CR14` (barrier), `CR15` (cache), `CR16` (cycles), `CR18` (`RASC_ID`), `CR 0x20..0xFF` banks, and exception handling |
| 11 | [rasc-toolchain-and-abi.md](rasc-toolchain-and-abi.md) | Freestanding compilation with `ARM-software/rasc-devkit` (GCC 4.2.3 + binutils 2.18 `rasc-elf`), measured ABI (`r0`–`r4`, `r0:r1`, `r5`–`r13`), linker script, and pitfalls |

### 4. Internal Core Hardware Programming (Boot, IRQs, Recovery, and Subsystems)

| # | Document | Contents |
|---:|---|---|
| 12 | [boot-core-init.md](boot-core-init.md) | Step-by-step core boot sequence: `CR3=0x1A7`, RASC1..4 vectors/resets (`0xFF023000`), spinlock `0xFF02B038`, `CR4`, `CR15`, defaults (`0xFF02400C`, `0xFF02B034`, `0xFF021F04`), `0xFF02B068..7C` validation, and `0xFF02B030` switch-out |
| 13 | [rasc-irq-and-sync.md](rasc-irq-and-sync.md) | 4-vector table at `CR0`, nestable `clri`/`seti`, 5 IRQ controllers (`0xFF026000..0xFF02A000`), `IRQHOST` bits (b18 + clear `0xFF02B02C`), `MMU` (b3), `WATCHDOG` (b4), inter-core (b23), doorbells, and atomic spinlock |
| 14 | [hardware-recovery.md](hardware-recovery.md) | Watchdog (`0xFF025000`), `MMU ABORT` across the 8 bus masters (`0xFF024010+`, stride `0x20`), `RASC ABORT`, SDMA fault (`0xFF021F00`), `DUMP`, MVE errors, and `TERMINATE` |
| 15 | [internal-mmio-map.md](internal-mmio-map.md) | Consolidated map of `0xFF00_0000`–`0xFF2F_8FFF`: Session SRAM, Core-Local SRAM, `SDMA`, `VDMA`, `RASC_CTRL`, `MMU`, `WATCHDOG`, 5×IRQ, `SYSCTRL`, and the 6 video blocks (`0xFF04_0000`..`0xFF18_0000`, 256 KiB stride) |
| 16 | [video-subsystems.md](video-subsystems.md) | Fixed subsystems (`BITSTREAM`, `REF`, `ME`, `MC`, `TRF`, `DBL`, `SDMA`, `VDMA`, `SYSCTRL`, `MMU`), decode/encode flows, `IntBuf`, DPB, and multi-core coordination |

### 5. Host ↔ Firmware Protocol (MVE 3.5)

| # | Document | Contents |
|---:|---|---|
| 17 | [mve-queues-and-rpc.md](mve-queues-and-rpc.md) | 6 circular 4 KiB queues (`0x1007_9000`–`0x1007_E000`, 1020 words), word-aligned packing, RPC channel (`0x1007_F000`), and circular print RAM (`0x1010_0000`, magic `0x11223356`) |
| 18 | [mve-messages-and-states.md](mve-messages-and-states.md) | Complete catalog of requests (`1001`–`1013`), responses (`2001`–`2019`), error codes (`1`–`9`), events (`PROCESSED`, `REF_FRAME`, `TRACE_BUFFERS`), and state machine (`STOPPED`/`RUNNING`) |
| 19 | [mve-options.md](mve-options.md) | Complete reference for `MVE_REQUEST_CODE_SET_OPTION` (`1010`): all indices (`1`–`69`), structured payloads, and bitmasks |
| 20 | [mve-buffers-and-formats.md](mve-buffers-and-formats.md) | Buffer descriptors (`3001`–`3004`): `mve_buffer_frame` (Planar and AFBC, `format` bitfield), `mve_buffer_bitstream`, `mve_buffer_param` (19 types), `mve_buffer_general`, and validations |

### 6. Codecs, Board Validation, and Reference

| # | Document | Contents |
|---:|---|---|
| 21 | [codecs-and-capabilities.md](codecs-and-capabilities.md) | Matrix of the 12 decode codecs and 5 encode codecs, `.fwb` files (including H.263 → `mpeg4dec.fwb`), pixel formats, fuses, and per-codec restrictions |
| 22 | [observed-sequences-and-gaps.md](observed-sequences-and-gaps.md) | Real sequences on K1 and K3 (H.264 128×128 decode, RPC, custom RASC firmware), platform pitfalls, and formal registry of open gaps (**G1–G5**) |
| 23 | [glossary.md](glossary.md) | Glossary of acronyms and technical terms for Linlon V5, RASC, MVE 3.5, AFBC, LSID, and SpacemiT K1/K3 architecture |

---

## Reimplementation Coverage Status

- **Confirmed silicon target:** Arm China **Linlon V5**, `HARDWARE_ID = 0x56635002` (family `0x5663`, revision `0x50` = 80, patch `0x02`), `FUSE = 0x00000000`, `NLSID = 4`.
  - **SpacemiT K1:** `NCORES = 2`, `CONFIG = 0xFFFF6402`, default clock `819.2 MHz`.
  - **SpacemiT K3:** `NCORES = 4`, `CONFIG = 0xFFFF7404`, default clock `1 GHz`.
- **Control Plane (Milestones M1, M2, and M3) — FULL COVERAGE:**
  1. `.fwb` container, MMU loading, trampoline, PC-relative stack, scalar RASC ISA, and GCC 4.2.3 ABI validated on silicon.
  2. Complete core boot sequence ([boot-core-init.md](boot-core-init.md)): RASC0 + RASC1..4 (`0xFF023000`), hardware spinlock (`0xFF02B038`), `CR4` instance base, `CR15` cache, `CR0` VBAR (4 vectors), unit defaults (`0xFF02400C`, `0xFF02B034`, `0xFF021F04`), `HARDWARE_ID`/DSL validation, and switch-out (`0xFF02B030`).
  3. Per-RASC interrupt controllers (`0xFF026000..0xFF02A000`), `IRQHOST` (bit 18 + clear `0xFF02B02C`), FW→Host IRQ (`CR14=0` + `0xFF02B028=1`), Watchdog (`0xFF025000`), and `MMU ABORT` across the 8 bus masters (`0xFF024010+`).
  4. Complete MVE 3.5 protocol (circular queues, RPC, messages, options, buffers, events, and `ram_print`).
- **Video Data Plane (Milestones M4–M6) — BASE MAPPED, PER-CODEC SEQUENCES OPEN:**
  - All MMIO base addresses of the 6 video accelerators (`0xFF04_0000`–`0xFF18_0000`, 256 KiB stride) and DMAs (`SDMA` `0xFF02_1000`, `VDMA` `0xFF02_2000`) + primary status/control registers are mapped.
  - Detailed bit-by-bit programming to trigger frames/stripes on each accelerator (gaps **G1–G2** in [observed-sequences-and-gaps.md](observed-sequences-and-gaps.md)) will be closed in dedicated data-plane rounds (starting with JPEG in M4 and H.264 in M5).

---

## License

This specification is distributed under the **Creative Commons Attribution-ShareAlike 4.0 International (`CC-BY-SA-4.0`)** license. See the [LICENSE](LICENSE) file for the full terms.

*(Note: public ABI definitions referenced from the Linux `mvx` driver and the public ISA table from `ARM-software/rasc-devkit` remain under their original `GPL-2.0-only` / `GPLv2` licenses.)*
