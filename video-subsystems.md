<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Internal Video Subsystems and Hardware Pipeline

This document describes the fixed-function video acceleration blocks present in each Linlon V5 VPU core, the architectural role of each block, the decoding and encoding dataflows, the intermediate buffer (`IntBuf`) model, DPB management, and multi-core coordination requirements.

---

## 1. Per-Core Hardware Subsystem Catalog (`[P][D]`)

Each Linlon V5 core combines **5 RASC CPUs** with the following fixed-function accelerator blocks, whose roles and control domains are established in the public Arm China Linlon V5/V7 architecture and reflected in the MVE 3.5 protocol options:

| Block | Full Name | Hardware Role | MVE-Controllable Features |
|---|---|---|---|
| **BITSTREAM** | Entropy / Bitstream Engine | Hardware-accelerated entropy parsing and coding: CABAC, CAVLC, VP8/VP9 boolean decoder, motion vector prediction, start-code search, and emulation prevention (*escaping*) handling | `NALU_FORMAT`, `STREAM_ESCAPING`, `ENC_H264_CABAC`, `ENC_CABAC_INIT_IDC`, `ENC_VP9_PROB_UPDATE`, `ENC_ENTROPY_SYNC` |
| **REF** | Reference & Cache Subsystem | Stores, indexes, and prefetches/caches reference frames (preferably in **AFBC** compressed format), with bandwidth management and size limits | `DISABLE_FEATURE_REF_CACHE`, `DISABLE_FEATURE_AFBC_LEGACY_REF`, `DISABLE_FEATURE_REF_PICS`, `DISABLE_FEATURE_REFSZ_LIMIT` |
| **ME** | *Motion Estimation* | Motion vector search during encode (hierarchical search / configurable X×Y window, MB/CU partition selection) | `MV_SEARCH_RANGE`, `ENC_MB_TYPE_ENABLE/DISABLE`, `ENC_MIN_LUMA_CB_SIZE` |
| **MC** | *Motion Compensation* | Sub-pixel interpolation and inter-frame prediction using references from the `REF` block | Decode pipeline and encode reconstruction |
| **TRF** | *Transform & Quantization* | Forward/inverse transforms (DCT/DST/integer transforms 4×4..32×32) and quantization/dequantization | `QUANT_TABLE` (JPEG), `QP`, `CB/CR_QP_OFFSET`, `ADAPTIVE_QUANTISATION` |
| **DBL** | *Deblocking & Loop Filter* | In-loop deblocking filter (H.264/HEVC/VP8/VP9/etc.) and **SAO** (*Sample Adaptive Offset*, HEVC) | `DISABLE_FEATURE_DEBLOCK`, `DISABLE_FEATURE_SAO`, `ENC_SAO_PENALTY/LUMA_EN/CHROMA_EN` |
| **SDMA** | *System DMA* | Transfers structured data between DRAM and local SRAM: `mbinfo`, residual coefficients, context rows (*row buffers*), and stripe metadata | Intermediate buffers (`IntBuf`), `MBINFO_OUTPUT` |
| **VDMA** | *Video DMA* | Transfers pixel stripes between DRAM (`FRAMEBUF`) and the pipeline: AFBC compression/decompression, rotation (90°/180°/270°), mirroring, RGB→YUV conversion, and **flexible downscaling** (DSL) | `DISABLE_FEATURE_AFBC_COMP`, `ROTATION_*`, `MIRROR_*`, `DEC_DOWNSCALE`, `ENC_RGB_TO_YUV_MODE` |
| **SYSCTRL** | Core System Control | Core hardware revision, internal clock gating / `SLEEP` control, and internal timing/watchdog | `DISABLE_FEATURE_SLEEP`, `WATCHDOG_TIMEOUT` |
| **MMU** | Memory Management Unit | Per-LSID 32-bit VA → 40-bit PA translation and access fault detection (`MMU ABORT`) | Host-controlled (`MMU_CTRL`, `FLUSH_ALL`) |

---

## 2. Dataflow — Decoding (`[P][D]`)

Decoding is decoupled into **two macro stages** (allowing one core or RASC CPU to parse frame *N+1* while another reconstructs frame *N*):

```text
  Bitstream in DDR (PROTECTED region)
       │
       ▼  (Bitstream DMA)
  ┌────────────────────────── STAGE 1: PARSE ────────────────────────────┐
  │  BITSTREAM (CABAC / CAVLC / VP8-VP9 bool / VLC) + RASC CPU           │
  │   • Decodes headers (SPS/PPS/VPS/Slice)                              │
  │   • Validates resolution, profile, bitdepth, chroma                  │
  │   • Extracts entropy symbols → produces mbinfo + residuals           │
  └──────────────────────────────────┬───────────────────────────────────┘
                                     │  SDMA
                                     ▼
                    IntBuf (DDR — allocated via RPC MEM_ALLOC)
                                     │  SDMA
                                     ▼
  ┌────────────────────────── STAGE 2: PIPE ─────────────────────────────┐
  │  Reconstruction by Stripes / Macroblocks / CTUs:                     │
  │                                                                      │
  │   REF (AFBC references) ──► MC ──► TRF (IQ/IT) ──► DBL (+ SAO)       │
  │             ▲                                            │           │
  │             └─────────── reference frame writeback ◄─────┤           │
  │                                                          ▼           │
  │                       VDMA (planar / AFBC / rotation / downscale)    │
  └──────────────────────────────────┬───────────────────────────────────┘
                                     │  128-bit AXI
                                     ▼
                   Output framebuffer in DDR (FRAMEBUF region)
```

### Behavioral Requirements for Decoding

1. **Parse ↔ Pipe Decoupling (`IntBuf`):**
   - The *parse* stage writes `mbinfo` and residual coefficients into intermediate buffers (`IntBuf`) dynamically allocated via RPC (`MVE_RPC_FUNCTION_MEM_ALLOC`, typically in the `PROTECTED` region).
   - Option `MVE_SET_OPT_INDEX_INTBUF_SIZE` (`9`) suggests a size limit for the `IntBuf` allocator.
   - Flag `MVE_OPT_DISABLE_FEATURE_PIPE` (`0x40`) allows operating in *parser-only* mode (diagnostics).
2. **DPB Management and Reordering:**
   - The firmware manages the **DPB (*Decoded Picture Buffer*)**: display order, reference frame marking (`is_ref`), *decode-only* frames (`MVE_BUFFER_FRAME_FLAG_DECODE_ONLY`, e.g., invisible VP9 frames or pre-roll), and draining on `EOS` or sequence change.
   - `MVE_SET_OPT_INDEX_FRAME_REORDERING` (`8`, default `1`) enables or disables output reordering in the DPB.
   - The firmware can inform the host how many output buffers are held in the DPB for reordering via `MVE_BUFFER_PARAM_TYPE_DPB_HELD_FRAMES` (`19`).
3. **AFBC References and Private Pools:**
   - When the output format requested by the host is **AFBC**, the output buffers supplied by the host can themselves act as reference frames (the firmware returns the buffer marked with `MVE_BUFFER_FRAME_FLAG_REF_FRAME`; the host must not overwrite it and must re-queue it after display, or send `MVE_REQUEST_CODE_RELEASE_REF_FRAME` and wait for `MVE_RESPONSE_CODE_REF_FRAME_UNUSED`).
   - When the output format requested by the host is **planar** (e.g., NV12/I420), the hardware still requires internal references (in AFBC); the firmware allocates a private reference pool via RPC (`MVE_MEM_REGION_OUTBUF`/`FRAMEBUF`) — which is why `sequence_parameters.num_buffers_afbc` is greater than `num_buffers_planar` (`[P]`).
4. **Dynamic Sequence Change:**
   - Upon decoding the first sequence header (or detecting a change in resolution, interlace, chroma format, or bitdepth), the firmware must emit `MVE_RESPONSE_CODE_FRAME_ALLOC_PARAM` followed by `MVE_RESPONSE_CODE_SEQUENCE_PARAMETERS` and enter a wait state until the host sends `MVE_REQUEST_CODE_OUTPUT_FLUSH`, drain all old output buffers, emit `MVE_RESPONSE_CODE_OUTPUT_FLUSHED`, and receive the newly reallocated buffers (`[P][O]`).

---

## 3. Dataflow — Encoding (`[P][D]`)

```text
  Input frame in DDR (YUV planar / semi-planar / AFBC / RGB)
       │
       ▼  VDMA (optional: RGB→YUV conversion, crop, rotation, mirror)
  ┌────────────────────────── ENCODE PIPELINE ───────────────────────────┐
  │                                                                      │
  │   REF (reconstructed frames / LTR) ◄───────────────────┐             │
  │             │                                          │             │
  │             ▼                                          │             │
  │   ME (motion search) ──► MC + Intra Pred               │             │
  │                               │                        │             │
  │                               ▼                        │             │
  │                      TRF (T/Q + IQ/IT) ──► DBL (+ SAO)               │
  │                               │                                      │
  │                               ▼                                      │
  │                BITSTREAM (CABAC / CAVLC / VP8-VP9 bool / JPEG)       │
  └──────────────────────────────────┬───────────────────────────────────┘
                                     │  Bitstream DMA
                                     ▼
                  Output bitstream buffer (PROTECTED region)
```

### Behavioral Requirements for Encoding

- **Rate Control and QP:** Supports fixed QP per frame type (`QP_I`, `QP_P`, `QP_B`), regions of interest (`MVE_BUFFER_PARAM_TYPE_REGIONS`, up to 16 macroblock regions), per-block/quad configuration (`MVE_BUFFER_GENERAL_TYPE_BLOCK_CONFIGS`), adaptive quantization (`ADPTIVE_QUANTISATION`), Q8 fixed-point lambda scales (`LAMBDA_SCALE`), and rate control modes (`OFF`, `STANDARD`, `VARIABLE`, `CONSTANT`, `C_VARIABLE`, with `qp_min`/`qp_max` and HRD buffer size).
- **GOP Structure and References:** Supports bidirectional, low-delay, and pyramid GOPs (`GOP_TYPE`), intra refresh (`INTRA_MB_REFRESH`), *Long Term Reference* (`ENC_LTR_MODE`/`PERIOD`), GOP reset (`MVE_BUFFER_PARAM_TYPE_GOP_RESET`), and change rectangles (`CHANGE_RECTANGLES`, up to 2 rectangles) for bandwidth savings.
- **Parallelism:** Configurable tiles (`TILES`: rows/columns), `ENC_ENTROPY_SYNC` (WPP in HEVC), `ENC_SLICE_SPACING`, and multi-core scaling (`ENC_MAX_NUM_CORES`).

---

## 4. Multi-Core Coordination and Bandwidth Management (`[P][D]`)

When a job is submitted with `cores > 1` (e.g., `cores = 2` on K1 or `cores = 4` on K3):

1. **Master / Slave Role Between Cores:**
   - One of the physical cores acts as the session coordinator (owner of the MVE queues and frame/slice distribution), while the others act as *slaves* executing assigned stripes/frames.
   - Shared session state resides in the **`master_rw`** region (common physical pages mapped at the same virtual offset across all core instances).
2. **Bandwidth Manager:**
   - Hardware/firmware can arbitrate DDR access to limit bandwidth peaks (`ENC_BANDWIDTH_LIMIT` in encode; `MVE_JOB_FLAG_DISABLE_BNDMGR = 0x01` in `JOB` disables the bandwidth manager).
3. **Accounting and Profiling (`MVE_SET_OPT_INDEX_PROFILING`):**
   - When enabled, the firmware collects *parse* and *pipe* start/end/idle timestamps, `parser_coreid`, `pipe_coreid`, bitstream bits, `IntBuf` size, total allocated memory, bus bytes read/written, and AFBC bytes, reporting them to the host via `MVE_EVENT_PROCESSED` (`3`).

---

## 5. Video Register Specification Status

The logical names and behaviors above are established by the public contract `[P]` and observations `[O]`. The **field-by-field** internal register map of `BITSTREAM`, `REF`, `ME`, `MC`, `TRF`, `DBL`, `SDMA`, and `VDMA` in the `0xFF00_0000`–`0xFF2F_8FFF` range remains open — see [internal-mmio-map.md](internal-mmio-map.md) and [observed-sequences-and-gaps.md](observed-sequences-and-gaps.md).

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
