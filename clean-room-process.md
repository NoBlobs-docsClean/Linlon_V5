<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Clean-Room Process and Information Firewall

This document defines the isolation protocol (*clean-room*) under which this specification was written and under which the clean team must produce the reimplementation of the **Arm China Linlon V5** (`HARDWARE_ID = 0x56635002`) VPU firmware for the **SpacemiT K1** and **SpacemiT K3** SoCs.

---

## 1. Separation of Roles

Development follows the classic two-team model isolated by a documentary firewall:

1. **Specification Team (dirty side → produces this folder)**
   - Inspects public sources (GPL-2.0 Linux `mvx` driver, public `ARM-software/rasc-devkit` toolchain, datasheets, and Device Trees), empirical captures on SpacemiT K1 and K3 boards, and observable hardware behavior.
   - Distills exclusively **hardware facts**, **public ABI contracts**, **state machines**, **ISA specifications**, and **functional register sequences** into `VPU/docsClean/`.
2. **Implementation Team (clean side → consumes this folder)**
   - Has never had access to proprietary `.fwb` blobs, disassemblies, decompiled/lifted code, or proprietary Arm China firmware source code.
   - Reads exclusively this specification, public open-source Linux driver headers (`GPL-2.0-only`), the public `ARM-software/rasc-devkit` toolchain (`GPLv2`), public video standards (ITU-T H.264, H.265/HEVC, JPEG, VP8, VP9, MPEG-2/4, etc.), and SpacemiT Device Trees/datasheets.
   - Writes the entire firmware implementation (RASC runtime, `crt0`, linker script, MVE 3.5 handler, internal scheduler, and video block controllers) from scratch, with its own internal architecture, private structures, symbol names, and algorithms.

---

## 2. What May and May Not Enter This Specification

### Allowed

- **Public contracts of the GPL-2.0 Linux driver (`mvx` / `vpu_k1x` / `vpu_k3`):**
  - Protocol header `mve_protocol_def.h`: virtual regions `MVE_MEM_REGION_*`, queue addresses `MVE_COMM_*`, structures `mve_comm_area_*`, `mve_rpc_communication_area`, `mve_msg_header`, codes `MVE_REQUEST_CODE_*`, `MVE_RESPONSE_CODE_*`, `MVE_BUFFER_CODE_*`, options `MVE_SET_OPT_INDEX_*`, formats `MVE_FORMAT_*`, buffer descriptors, and events.
  - Host-side APB registers (`mvx_hwreg.h`/`mvx_hwreg.c`), fields of `JOBQUEUE`, `CORELSID`, `CTRL`, `FUSE`, and LSID sequence (`mvx_lsid.c`).
  - 2-level MMU format (`mvx_mmu.h`/`mvx_mmu.c`), 32-bit PTE layout, and K1 physical DMA translation (`phys_cpu2vpu`/`phys_vpu2cpu`).
  - `.fwb` container header (`struct mvx_fw_header`, 180 bytes) and driver loading/mapping rules (`mvx_firmware.c`, `mvx_firmware_cache.c`).
- **Public RASC ISA and ABI specification (`ARM-software/rasc-devkit`):**
  - Instruction encoding table, bitmasks, immediate and branch suffixes, BCC3 semantics, register set (`r0`–`r15`, `v0`–`v15`), and `rasc-elf-gcc` calling convention.
- **Empirical hardware facts measured on silicon (K1 and K3):**
  - Values read from global/LSID registers, clock frequencies, MVE 3.5 protocol traces, and results from custom probes executed on the K1's RASC CPUs.
- **Internal hardware registers without public documentation:**
  - When an internal MMIO register (`0xFF00_0000`–`0xFF2F_8FFF`) or control register (`CR0`–`CR255`) is required to operate the silicon (e.g., `0xFF02B028` to signal an IRQ to the host, `CR14` for synchronization, `CR15` for cache maintenance), it is specified exclusively by its **address/number**, **access width**, **direction (R/W)**, **required value/mask**, and **observable hardware effect**, explicitly marked as having no public documentation.
- **Behavioral requirements and state machines:**
  - Preconditions, postconditions, invariants, buffer/option validations, MVE state transitions, and data flows between hardware blocks (`BITSTREAM`, `REF`, `ME`, `MC`, `TRF`, `DBL`, `SDMA`, `VDMA`).

### Strictly Forbidden

- Snippets of C code, RASC assembly, or pseudocode copied from proprietary `.fwb` blobs or from any disassembly/decompilation/lifting tool.
- Internal code, table, data, BSS, or stack addresses from proprietary blobs (for example, internal function offsets, internal table pointers, or private `master_rw` layouts from the original firmware).
- Internal function names, private variables, internal diagnostic strings copied literally as a requirement, or private layouts of internal structures not exposed in the public `mve_protocol_def.h` contract.
- Statistics, hashes, SVN revisions, BSS bitmap tables, or build artifacts extracted from proprietary Arm China binaries.

---

## 3. Source Hierarchy and Provenance

In case of doubt or apparent divergence between sources, apply the following precedence:

1. **`[P]` Normative public sources**
   - GPL-2.0 Linux driver (`mve_protocol_def.h`, `mvx_hwreg.*`, `mvx_lsid.*`, `mvx_mmu.*`, `mvx_firmware*.*`), Device Trees (`k1-x.dtsi`, K3 DT), and `ARM-software/rasc-devkit` (`rasc-opc.c`, `rasc-dis.c`, `gcc/config/rasc/` backend).
2. **`[O]` Evidence measured on real hardware (SpacemiT K1 BPI-F3 and K3 Pico ITX)**
   - Actual behavior of `0x56635002` silicon: takes precedence over documentation inferences (for example, silicon-measured semantics of `movh = <<1`, `movw = <<2`, `movn = ~imm`, and 8-bit CRs, as well as identical `HARDWARE_ID = 0x56635002` between K1 and K3).
3. **`[D]` Specification derived from silicon interaction**
   - Initialization sequences, internal MMIO addresses, and operational semantics of `CR`s required for the hardware to satisfy `[P]` and `[O]`.

---

## 4. Clarification Protocol (Question Loop)

If during reimplementation the clean team encounters a hardware gap or ambiguity:

1. **Log an objective question focused on a contract or hardware register**, for example:
   - *"Which bits of the internal register at `0xFF0230xx` control the reset release of each worker RASC CPU (RASC1..4)?"*
   - *"What is the register programming sequence for the `VDMA` block to transfer an NV12 frame from the `FRAMEBUF` virtual buffer to internal SRAM?"*
   - *"Which bit of the Status Register (`SR`) enables external interrupts when executing `seti`/`clri`?"*
2. **The specification team investigates the hardware/experiment and updates the corresponding document in `docsClean/`** with register tables, sequences, or behavioral rules — never sending code or internal implementation details of the proprietary firmware.
3. **Never assume unverified details from other Mali-V / Linlon IPs** (such as Mali-V500/V550/V61 or other generations): internal MMIO offsets, fuses, flexible downscaling features, and supported formats vary by silicon family.

---

## 5. Clean-Room Firmware Acceptance Criteria

A clean-room reimplementation is considered functionally progressive against the `mvx` driver (K1 and K3) according to the verifiable milestones below:

| Milestone | Verifiable Requirement |
|---|---|
| **M1 — `.fwb` Container, RASC Boot, and MVE Channel** | Generates a valid `.fwb` (180 B header + reset trampoline + `text_length` including header + PC-relative stack), boots on RASC0 of K1/K3, writes a response to the `MSG_OUTQ` queue (`0x1007_A000`), executes `CR14 = 0` + `0xFF02B028 = 1`, and the Linux driver receives the IRQ and decodes the message without protocol errors. |
| **M2 — Complete MVE 3.5 Handshake (Control Plane)** | Responds with `SWITCHED_IN`, processes `PING`→`PONG`, `SET_OPTION`→`SET_OPTION_CONFIRM`/`FAIL`, `JOB`→`JOB_DEQUEUED`, `GO`→`STATE_CHANGE(RUNNING)`, `STOP`→`STATE_CHANGE(STOPPED)`, `INPUT_FLUSH`/`OUTPUT_FLUSH`, `IDLE`/`IDLE_ACK`, `SWITCH`→`SWITCHED_OUT`, and requests memory via RPC (`MEM_ALLOC`/`MEM_RESIZE`/`MEM_FREE`) without state-machine violations. |
| **M3 — Multi-RASC and Multi-Core (K1 2 cores / K3 4 cores)** | Properly initializes the 5 RASC CPUs per core (worker vectors, cache via `CR15`, exceptions via `FAR`/`PC`/`SR`) and coordinates multiple core instances via `master_rw` when scheduled with `cores = 2` (K1) or `cores = 4` (K3). |
| **M4 — First Functional Codec (JPEG / MJPEG)** | Decodes (and/or encodes) JPEG Baseline frames via the `BITSTREAM`/`TRF`/`VDMA` blocks, delivering bit-exact or ITU-T T.81-compliant planar output (NV12/I420) in the V4L2 M2M driver. |
| **M5 — Complete H.264 / HEVC Decode** | Executes full bitstream parsing (`BITSTREAM`), intermediate buffers (`IntBuf` via RPC), DPB with reordering and references (`REF`/AFBC), `MC` + `TRF` + `DBL` + `VDMA`, emitting `FRAME_ALLOC_PARAM` and `SEQUENCE_PARAMETERS`, supporting dynamic resolution switching and NV12/I420/AFBC output up to 4K. |
| **M6 — Codec Suite, Encode, and Robustness** | Progressive support for encode (H.264/HEVC/VP8/VP9/JPEG with ME and rate control), remaining decode codecs (VP8/VP9/MPEG-2/MPEG-4/H.263/VC-1/RV/AVS/AVS2), concurrent multi-session across 4 LSIDs, watchdog, and clean recovery on corrupted streams. |

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
