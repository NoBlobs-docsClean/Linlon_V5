<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Observed Real-Hardware Sequences, Pitfalls, and Open Gaps

This document records empirical evidence collected on real boards (**SpacemiT K1 BPI-F3** and **SpacemiT K3 Pico ITX**), platform pitfalls discovered during clean-room firmware bring-up on the RASC CPUs, and the updated catalog of **hardware gaps still open** for the advanced video acceleration milestones (M4–M6).

---

## 1. Real H.264 Decode Session Trace on SpacemiT K1 (`[O]`)

Captured via `debugfs` (`amvx/log/drain/ram0/msg`) on SpacemiT K1 using a **128×128, 5-frame (3367 bytes)** H.264 Baseline Annex-B stream, **NV12** output (4 capture buffers), producing 122,880 bytes (`5 × 128 × 128 × 1.5`) without errors:

```mermaid
sequenceDiagram
    participant H as Host (Linux mvx driver)
    participant F as VPU Firmware (RASC)

    Note over H: Loads h264dec.fwb, maps MMU (2 cores), LSID 0, JOBQUEUE
    H->>F: SET_OPTION(42: WATCHDOG_TIMEOUT = 0)
    H->>F: JOB(cores=2, frames=1, flags=0)
    H->>F: SET_OPTION(1: NALU_FORMAT = 1 [start codes])
    H->>F: SET_OPTION(5: PROFILING = 1)
    H->>F: GO
    H->>F: PING
    H->>F: BUF_BITSTREAM(alloc=16384, fill=3367, addr=0x20000000)
    H->>F: 4 × BUF_FRAME (NV12 output)

    F-->>H: RPC MEM_ALLOC (×3, e.g., 128 KiB, max 64 MiB, PROTECTED, align 4 MiB → VA 0x20400000)
    F-->>H: SWITCHED_IN(core=0)
    F-->>H: SET_OPTION_CONFIRM (×3: indices 42, 1, 5)
    F-->>H: PONG
    F-->>H: JOB_DEQUEUED(valid_job=1)
    F-->>H: STATE_CHANGE(new_state=2 [RUNNING])
    F-->>H: FRAME_ALLOC_PARAM(planar=128x128, afbc=53248, afbc_ds=28672, w_sb=8/4, crop=0,4, mbi=114688, ctu=16)
    F-->>H: SEQUENCE_PARAMETERS(interlace=0, chroma=1 [4:2:0], bitdepth=8/8, planar_bufs=1, afbc_bufs=4)

    H->>F: OUTPUT_FLUSH
    F-->>H: IDLE
    H->>F: IDLE_ACK
    F-->>H: 4 × OUTPUT (returns old buffers via BUF_OUTRQ)
    F-->>H: OUTPUT_FLUSHED

    F-->>H: RPC MEM_ALLOC (×4, additional buffers / references)
    F-->>H: INPUT (bitstream returned: fill=0, offset=3367)
    F-->>H: IDLE
    H->>F: SWITCH
    F-->>H: SWITCHED_OUT(core=0, reason=1, sub_reason=0)

    H->>F: STOP
    H->>F: INPUT_FLUSH
    F-->>H: STATE_CHANGE(new_state=0 [STOPPED])
    F-->>H: INPUT_FLUSHED
    H->>F: OUTPUT_FLUSH
    F-->>H: OUTPUT_FLUSHED
```

### Behavioral Facts Confirmed by This Trace (`[O]`)

1. **Initial batch already queued at boot:** The driver enqueues `SET_OPTION`, `JOB`, `GO`, `PING`, and buffers before even receiving the first `SWITCHED_IN`.
2. **Resolution-independent multi-core scheduling:** The driver requested `cores = 2` even for 128×128; the firmware accepted and completed the decode.
3. **Large alignments in RPC `MEM_ALLOC`:** Typical requests use `size = 128 KiB`, `max_size = 64 MiB`, `region = 0 (PROTECTED)`, `log2_alignment = 22 (4 MiB)` (returned VA `0x2040_0000`).
4. **Post-`SEQUENCE_PARAMETERS` reallocation handshake:** The host always sends `OUTPUT_FLUSH` after receiving `FRAME_ALLOC_PARAM` + `SEQUENCE_PARAMETERS`.
5. **Idle switch-out:** `IDLE` → `SWITCH` → `SWITCHED_OUT(core=0, reason=1, sub_reason=0)`, followed by `0xFF02B030 = 1`, returning `CORELSID` to `0x0000_00FF` (K1).

---

## 2. Clean-Room Firmware Validation on K1 Silicon (`[O]`)

A minimal firmware built from scratch with `rasc-elf-gcc` 4.2.3 + binutils 2.18 executed directly on SpacemiT K1's RASC0 CPU and exchanged valid MVE messages with the Linux `mvx` driver:

| Silicon Test (K1) | Observed Result | Implication for the Clean Team |
|---|---|---|
| `_start` execution via `br` trampoline at offset `0x00` | Success (`VA 0x1000` jumps over the 180 B header) | Mandatory to place `br _start` in `rasc_jmp` and include the 180 B in `text_length` |
| Stack initialization with `lpc sp, stack_top` | Success on any core instance | Forbidden to use a fixed absolute address for `sp` |
| Scalar arithmetic and shifts (`movh`, `movw`, `movn`, `add`/`adc`) | `movh = <<1`, `movw = <<2`, `movn = ~imm`, carry functional | Use these exact semantics |
| `push`/`pop` of `r13..r14` | Restored correctly (`rN` at lowest address, `r14` at highest) | Compatible with `rasc-elf-gcc` and exception frame |
| BCC3 skipping 2 instructions of 3 bytes each | Skipped exactly 2 instructions (6 bytes) | BCC3 counts variable-length instructions, not multiples of 2 bytes |
| 8-bit CR number (`mtcr`/`mfcr`) | `CR 0x94` ≠ `CR 4` on silicon | Never truncate CR number to 4 bits |
| Sending `MVE_RESPONSE_CODE_ERROR` (`2010`) via `MSG_OUTQ` + `CR14 = 0` + `0xFF02B028 = 1` | Linux driver received IRQ 74, read `MSG_OUTQ`, and displayed the payload intact in `dmesg` | FW→Host MVE transport 100% proven on silicon |

---

## 3. Critical Platform Pitfalls (Lessons Learned) (`[O]`)

1. **`text_length` omitting the 180 header bytes silently truncates the firmware.**
2. **Mismatch between the `br` trampoline target (offset `0x00`) and the link offset of `_start`** causes the CPU to jump into the middle of code.
3. **Writes to extended CRs (`0x90`/`0x93`/`0x94`) and coherency:** Avoid writing to uncharacterized coprocessor CRs during scalar bring-up (cache/state side effects observed on K1).
4. **2 GiB DMA translation on K1 (`phys_cpu2vpu`):** Physical pages above `0x1_0000_0000` (CPU) require subtracting `0x8000_0000` in MMU PTEs.
5. **V4L2 process stuck in `D` state:** If experimental firmware does not respond, the session only releases after the driver's `watchdog_timeout` (default 30 s) or `TERMINATE`/block reset.
6. **`rasc-elf-as` 2.18 bug:** Never pass literal numeric addresses to `bsr` or `lpc` in `.S` files (causes segfault); always use symbolic labels.

---

## 4. Formal Register of Open Hardware Gaps

The **control-plane** gaps (multi-RASC boot, `CR0` VBAR, `CR4` instance base, `CR16` cycle counter, `CR18` `RASC_ID`, `clri`/`seti`, spinlock `0xFF02B038`, `IRQHOST` bit 18 + clear `0xFF02B02C`, switch-out `0xFF02B030`, watchdog `0xFF025000`, 8 MMU bus masters `0xFF024010+`, `HARDWARE_ID` / internal revision / DSL capabilities at `0xFF02B068..7C`, and the base addresses of all 6 video blocks `0xFF04_0000`..`0xFF18_0000`) **are already closed** in this specification (Milestones **M1, M2, and M3** are ready for implementation!).

The following items remain open for future specification rounds (targeting Milestones **M4–M6**):

| ID | Area | What Remains to Be Specified | Impact / Milestone |
|---|---|---|---|
| **G1** | Data Plane — Video Block Programming | Exact step-by-step register programming sequence (offsets, bitfields, and completion polls) inside `BITSTREAM` (`0xFF04_0000`), `REF` (`0xFF08_0000`), `ME` (`0xFF0C_0000`), `MC` (`0xFF10_0000`), `TRF` (`0xFF14_0000`), `DBL` (`0xFF18_0000`), `SDMA` (`0xFF02_1000`), and `VDMA` (`0xFF02_2000`) to execute a complete frame (starting with JPEG in M4 and H.264 in M5) | **M4–M6** (does not block M1–M3) |
| **G2** | Coprocessor CR Banks (`0x20`–`0xFF`) | Individual semantics of the extended control register banks coupled to the accelerators (`CR 0x20..0xFF`), including the cache/state side effects observed in `CR 0x90`/`0x93`/`0x94` | **M4–M6** |
| **G3** | `SR` Bits and Auxiliary IRQ Lines | Complete mapping of all *Status Register* (`SR`) bits, exact meaning of each bit in `CR3 = 0x1A7`, `CR6 = 0x200`/`1`, and mapping of auxiliary IRQ lines from video accelerator blocks (beyond bits 1, 3, 4, 17, 18, and 23) in controllers `0xFF02_6000..0xFF02_A000` | **M3–M5** (does not block M1–M2) |
| **G4** | Host Registers `CONFIG`, `PROTCTRL`, `INTSIG` | Bit-by-bit meaning of `CONFIG` (`0xFFFF6402` K1 vs. `0xFFFF7404` K3), `PROTCTRL` (`0x1`), and `INTSIG` | Low priority (Linux driver already manages them) |
| **G5** | ISA — `rrx` and Vector Extension `v*` | Exact behavior of `rrx` and per-lane semantics of vector instructions `v*`/`vf*` | Non-blocking (not used by GCC or in the control plane) |

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
