<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Overview — Arm China Linlon V5 VPU (SpacemiT K1 and K3)

This document presents the high-level architecture of the VPU integrated into the **SpacemiT Key Stone K1** and **SpacemiT K3** SoCs, the relationship between hardware variants, and the division of responsibilities between the host driver (Linux) and the firmware running on the embedded CPUs.

---

## 1. Hardware IP Identification (`[P][O]`)

| Attribute | Value | Provenance |
|---|---|---|
| **IP Vendor** | Arm Technology (China) Co., Ltd. | `[P]` |
| **Commercial Name / Line** | **Linlon V5** (driver family `MVE_v52_v76`, derived from the Arm Mali-V52/V76 lineage) | `[P][O]` |
| **`HARDWARE_ID` (offset `0x00`)** | **`0x56635002`** (family `0x5663`, revision `0x50` = 80, patch `0x02`) — identical on K1 and K3 | `[O]` |
| **`compatible` (Device Tree)** | `"arm china,linlon-v5"` (K1 and K3) | `[P][O]` |
| **Expected firmware directory** | `linlon-v52_v76-80-2/<codec><dec\|enc>.fwb` | `[P]` |
| **Host ↔ Firmware Protocol** | **MVE 3.5** (header `protocol_major = 3`, `protocol_minor = 5`) | `[P][O]` |
| **Embedded processor** | **5 × RASC CPUs** per VPU core (32-bit RISC, 5-stage pipeline, little-endian) | `[P]` |
| **Session contexts (LSID)** | **4 LSIDs** (`NLSID = 4`) on K1 and K3 | `[O]` |
| **Fuses (`FUSE`, offset `0x34`)** | `0x00000000` (all features enabled: AFBC, RealVideo, VP8/VP9, HEVC) | `[O]` |
| **Bus interfaces** | 128-bit AXI (data/DDR) + APB3 (host control registers) | `[P]` |
| **Framebuffer compression** | Native support for **AFBC** (Arm Frame Buffer Compression) 8-bit and 10-bit | `[P]` |

> **Critical note on naming (`[P][O]`):** The Linux driver maps both family `0x5662` and `0x5663` to the same internal identifier `MVE_v52_v76` (`0x5276`). **Neither K1 nor K3 is V76 (8K-class):** both report `HARDWARE_ID = 0x56635002` and `compatible = "arm china,linlon-v5"`, being **4K-class** configurations of the same Linlon V5 generation.

---

## 2. Structural Comparison: SpacemiT K1 vs. SpacemiT K3 (`[P][O]`)

Both SoCs instantiate the **same VPU IP**, differing in core count, clock frequency, and the value of the `CONFIG` topology register:

| Parameter | SpacemiT K1 | SpacemiT K3 | Impact on Firmware |
|---|---|---|---|
| `HARDWARE_ID` | `0x56635002` | `0x56635002` | None (same IP and revision) |
| `NCORES` | **2** | **4** | Transparent: the host instantiates the same binary per core and passes `cores` in the `JOB` message |
| `NLSID` | 4 | 4 | None |
| `CORELSID` (idle) | `0x000000FF` (2 nibbles) | `0x0000FFFF` (4 nibbles) | Managed only by the host driver |
| `CONFIG` | `0xFFFF6402` | `0xFFFF7404` | Read only by the host (topology difference) |
| Operating clock at probe | **819.2 MHz** | **1 GHz** | None: the firmware does not control the clock |
| Host MMIO base | `0xC050_0000` | `0xC050_0000` | Host side only |
| DT `reg` window | `64 KiB` (`0x10000`) | `8 KiB` (`0x2000`) | Host side only |
| IRQ | `74` (`0x4A`) | `74` (`0x4A`, level high) | Host side only |
| Nominal capability (Decode) | 4K@60 (H.264/HEVC) | 4K@180 | Scales linearly with `NCORES` × clock |
| Nominal capability (Encode) | 4K@30 (H.264/HEVC) | 4K@90 | Scales linearly with `NCORES` × clock |

**Fundamental architectural rule (`[P][O]`):** A single `.fwb` firmware binary per `(codec, direction)` must work **without recompilation** on both K1 (2 cores) and K3 (4 cores), provided that it:
1. Uses **PC-relative** addressing (`lpc`) to access its text and private BSS (since each core instance is mapped at `core * 0x0100_0000`).
2. Dynamically accepts the number of cores (`1..NCORES`) received in the `MVE_REQUEST_CODE_JOB` message (respecting that the core count is fixed for the duration of a job).
3. Does not depend on absolute clock frequencies for functional correctness (only for time/watchdog counters).

---

## 3. Internal Architecture of a Linlon V5 Core (`[P][D]`)

Each physical VPU core is neither a monolithic hardwired accelerator nor a software pixel DSP: it is a **firmware-controlled heterogeneous complex**, where **5 RISC microcontrollers (RASC CPUs)** manage fixed video processing blocks and DMAs.

**Core design principle (`[P]`):** *"The RASC CPUs never touch pixels directly — they manage the hardware subsystems."*

```text
┌────────────────────────────── Linlon V5 VPU (Top-Level) ──────────────────────────────┐
│  APB Interface (0xC050_0000) ──► Global Scheduler (JOBQUEUE, CORELSID, 4 × LSID)      │
│  128-bit AXI Interface       ◄── Bus Arbiter ◄── MMU (per-LSID, 2-level)              │
│                                                                                       │
│  ┌────────────────────── Core 0 ──────────────────────┐  ┌──── Core 1 (..3 on K3) ──┐ │
│  │  5 × RASC CPUs (32-bit, 5-stage RISC)              │  │  5 × RASC CPUs           │ │
│  │    [RASC0 Master] [RASC1] [RASC2] [RASC3] [RASC4]  │  │  REF / ME / MC / TRF /   │ │
│  │                                                    │  │  DBL / BITSTREAM /       │ │
│  │  Video Subsystems:                                 │  │  SDMA / VDMA / SRAM      │ │
│  │    BITSTREAM │ REF (AFBC) │ ME │ MC │ TRF │ DBL    │  │                          │ │
│  │                                                    │  │                          │ │
│  │  Data Movement and Control:                        │  │                          │ │
│  │    SDMA │ VDMA │ SYSCTRL │ Local MMU │ Local SRAM  │  │                          │ │
│  └────────────────────────────────────────────────────┘  └──────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

### Components of Each Core

1. **5 RASC CPUs (32-bit):** Execute the `.fwb` firmware mapped by the MMU. They have an internal MMIO space (`0xFF00_0000`–`0xFF2F_8FFF`) to program all video blocks and signal interrupts to the host.
2. **6 Video Subsystems:**
   - **BITSTREAM:** Entropy (CABAC, CAVLC, VP8/VP9 boolean decoding), accelerated parsing, and motion vector prediction.
   - **REF:** Management and caching of reference frames (including AFBC).
   - **ME:** Motion Estimation (used in encode).
   - **MC:** Motion Compensation.
   - **TRF:** Forward/inverse transform and quantization/dequantization.
   - **DBL:** Deblocking filter (and SAO in HEVC).
3. **2 DMA Engines:**
   - **SDMA:** System DMA for metadata, rows, `mbinfo`, and intermediate buffers.
   - **VDMA:** Video DMA for pixel stripes (planar, semi-planar, packed RGB/YUV, and AFBC).
4. **Per-Session MMU (LSID):** 2-level 32-bit VA → 40-bit PA translation (4 KiB pages), completely isolating each session.
5. **Multi-Core / Multi-Session Scheduler:** Allows up to 4 hardware-resident sessions (LSIDs 0–3) competing for physical cores via `JOBQUEUE`.

---

## 4. Responsibility Boundary: Host vs. Firmware (`[P]`)

The MVE 3.5 architecture enforces a strict layer separation. The firmware **never knows about V4L2, fourccs, or SoC power policies**; the host **never performs bitstream parsing, DPB management, or video block programming**.

| Responsibility | Host Side (Linux `mvx` Driver) | Firmware Side (RASC CPUs) |
|---|---|---|
| User API | V4L2 Mem2Mem (`/dev/video*`), V4L2 controls, `dmabuf` | Not visible (sees only the MVE 3.5 protocol) |
| Clock, Reset, Power, DVFS | Configures `CLK_VPU`, `RESET_VPU`, `PMU`, `RESET`/`CLKFORCE` | Has no access to CCU/PMU |
| Inter-session scheduling | Maps LSIDs (`0..3`), writes `JOBQUEUE`, `CTRL`, `SCHED`, sends `SWITCH` | Runs until idle or receiving `SWITCH`; saves context and responds with `SWITCHED_OUT` |
| MMU and Physical Memory | Allocates pages, populates L1/L2 tables, applies `phys_cpu2vpu`, writes `MMU_CTRL` and `FLUSH_ALL` | Uses exclusively 32-bit VAs; requests additional memory from the host via RPC (`MEM_ALLOC`/`RESIZE`/`FREE`) |
| Firmware Loading | Reads `.fwb`, validates header, maps text (RO/X) + private BSS + shared `master_rw` into each instance | Executes starting at `VA 0x1000` (trampoline at offset 0), initializes PC-relative stack and workers |
| Bitstream Parsing | Sends raw buffers (`mve_buffer_bitstream`) with flags (`EOS`, `CODECCONFIG`, etc.) | Decodes headers (SPS/PPS/VPS/slices), detects sequence changes, emits `FRAME_ALLOC_PARAM` and `SEQUENCE_PARAMETERS` |
| DPB and Reference Management | Provides frame buffers (`mve_buffer_frame`), honors `REF_FRAME` flag, may send `RELEASE_REF_FRAME` | Decides reordering, active references, private AFBC banks, discard (`DECODE_ONLY`), and `REF_FRAME_UNUSED` |
| Video Pipeline | — | Programs and synchronizes `BITSTREAM`, `REF`, `ME`, `MC`, `TRF`, `DBL`, `SDMA`, `VDMA` in stripes |
| Diagnostics and Errors | Reads `IRQVE`/`LIRQVE`, drains queues, reads `ram_print`, maintains session watchdog | Emits `ERROR`, `EVENT` (`PROCESSED`, `STREAM_CORRUPT`, etc.), writes to `ram_print`, and executes `DUMP` |

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
