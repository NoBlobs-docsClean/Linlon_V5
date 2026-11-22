<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# MVE 3.5 Protocol — Session Options (`SET_OPTION`)

This document lists all indices, payload structures, enumerations, and bitmasks supported by the `MVE_REQUEST_CODE_SET_OPTION` (`1010`) request, according to the public `mve_protocol_def.h` contract (`[P]`).

---

## 1. General Format and Semantics (`[P]`)

```c
struct mve_request_set_option {
    uint32_t index; /* option identifier (1..69) */
    union {
        uint32_t arg;
        /* or one of the typed structures described in §3 */
    } data;
};
```

### Firmware Processing Rules

1. **Required state:** Only accepted when the session is in `MVE_STATE_STOPPED` (`0`). In `RUNNING`, respond with `MVE_RESPONSE_CODE_SET_OPTION_FAIL` (`2017`).
2. **Confirmation:**
   - Success → send `MVE_RESPONSE_CODE_SET_OPTION_CONFIRM` (`2003`) containing `{ uint32_t index }`.
   - Invalid parameter, out of range, or not applicable to the codec → send `MVE_RESPONSE_CODE_SET_OPTION_FAIL` (`2017`) containing `{ uint32_t index; char message[128] }`. **Non-fatal error:** the session remains active and the option has no effect.

---

## 2. Complete Index Catalog (`MVE_SET_OPT_INDEX_*`) (`[P]`)

### 2.1 Common and Decoding Options

| Index | Name (`MVE_SET_OPT_INDEX_*`) | Scope | Field in `data` | Values / Default |
|---:|---|---|---|---|
| `1` | `NALU_FORMAT` | Dec/Enc | `arg` | `1`=START_CODES, `2`=ONE_NALU_PER_BUFFER, `4`=1B_LEN, `8`=2B_LEN, `16`=4B_LEN, `32`=MBINFO (debug) |
| `2` | `STREAM_ESCAPING` | Dec/Enc | `arg` | `1`=enabled (default), `0`=disabled |
| `3` | `PROFILE_LEVEL` | Dec/Enc | `profile_level` | `{ uint16_t profile; uint16_t level; }` (see §3.1) |
| `4` | `HOST_PROTOCOL_PRINTS` | Common | `arg` | `1`=enables protocol logs, `0`=disabled (default) |
| `5` | `PROFILING` | Common | `arg` | `1`=enables `EVENT(PROCESSED)`/profiling, `0`=disabled (default) |
| `6` | `DISABLE_FEATURES` | Common | `arg` | Bitmask `MVE_OPT_DISABLE_FEATURE_*` (see §3.2) |
| `7` | `IGNORE_STREAM_HEADERS` | Decode | `arg` | `1`=ignores stream headers, `0`=normal (default) |
| `8` | `FRAME_REORDERING` | Decode | `arg` | `1`=reorders output in DPB (default), `0`=disabled |
| `9` | `INTBUF_SIZE` | Decode | `arg` | Suggested limit in bytes for `IntBuf` allocation |
| `42` | `WATCHDOG_TIMEOUT` | Common | `arg` | Internal watchdog timeout; `0` = disables watchdog |
| `57` | `DEC_DOWNSCALE` | Decode | `downscaled_frame` | `{ uint16_t width; uint16_t height; }` — arbitrary downscale (DSL) |
| `69` | `DEC_DOWNSCALE_POS_MODE` | Decode | `dsl_pos` | `{ uint32_t mode; }` — DSL sampling positioning mode |

### 2.2 Encoding Options

| Index | Name (`MVE_SET_OPT_INDEX_*`) | Codec | Field in `data` | Description |
|---:|---|---|---|---|
| `16` | `ENC_P_FRAMES` | Enc | `arg` | Number of P-frames (`nPFrames`) between I-frames |
| `17` | `ENC_B_FRAMES` | Enc | `arg` | Number of consecutive B-frames |
| `18` | `GOP_TYPE` | Enc | `arg` | `1`=BIDIRECTIONAL, `2`=LOW_DELAY, `3`=PYRAMID |
| `19` | `INTRA_MB_REFRESH` | Enc | `arg` | *Intra macroblock refresh* parameter |
| `20` | `ENC_CONSTR_IPRED` | Enc | `arg` | `0`/`1`: *constrained intra prediction* |
| `21` | `ENC_ENTROPY_SYNC` | HEVC Enc | `arg` | `0`/`1`: *Wavefront Parallel Processing (WPP)* |
| `22` | `ENC_TEMPORAL_MVP` | HEVC Enc | `arg` | `0`/`1`: temporal MV prediction |
| `23` | `TILES` | HEVC/VP9 | `tiles` | `{ uint16_t tile_rows; uint16_t tile_cols; }` (HEVC: `tile_cols=0`; VP9: powers of 2) |
| `24` | `ENC_MIN_LUMA_CB_SIZE` | HEVC Enc | `arg` | `8` (8×8) or `16` (16×16) |
| `25` | `ENC_MB_TYPE_ENABLE` | Enc | `arg` | `MVE_MBTYPE_*` bitmask to enable (see §3.3) |
| `26` | `ENC_MB_TYPE_DISABLE` | Enc | `arg` | `MVE_MBTYPE_*` bitmask to disable (see §3.3) |
| `27` | `ENC_H264_CABAC` | H.264 Enc | `arg` | `0`=CAVLC, `1`=CABAC (default `1`) |
| `28` | `ENC_SLICE_SPACING` | Enc | `arg` | Suggested number of CTUs/MBs per slice |
| `30` | `ENC_VP9_PROB_UPDATE` | VP9 Enc | `arg` | `0`=DISABLED, `1`=IMPLICIT, `2`=EXPLICIT |
| `31` | `RESYNC_INTERVAL` | JPEG Enc | `arg` | *Restart interval* (`nRestartInterval`) |
| `32` | `HUFFMAN_TABLE` | JPEG Enc | `huffman_table` | Custom Huffman table (see §3.4) |
| `33` | `QUANT_TABLE` | JPEG Enc | `quant_table` | 8×8 quantization matrix (see §3.4) |
| `34` | `ENC_EXPOSE_REF_FRAMES` | Enc (debug) | `arg` | `0`/`1`: emits `MVE_EVENT_REF_FRAME` (default `0`) |
| `35` | `MBINFO_OUTPUT` | Enc (debug) | `arg` | `0`/`1`: MBInfo output (forbidden in protected session) |
| `36` | `MV_SEARCH_RANGE` | Enc | `motion_vector_search_range` | `{ int32_t mv_search_range_x; int32_t mv_search_range_y; }` |
| `38` | `ENC_STREAM_BITDEPTH` | Enc | `bitdepth` | `{ uint16_t luma_bitdepth; uint16_t chroma_bitdepth; }` |
| `39` | `ENC_STREAM_CHROMA_FORMAT` | Enc | `arg` | Encoded stream chroma format (`MVE_CHROMA_FORMAT_*`) |
| `40` | `ENC_RGB_TO_YUV_MODE` | Enc | `arg` | `0`=BT601_STUDIO, `1`=BT601_FULL, `2`=BT709_STUDIO, `3`=BT709_FULL |
| `41` | `ENC_BANDWIDTH_LIMIT` | Enc | `arg` | Host-defined maximum bandwidth limit |
| `43` | `ENC_CABAC_INIT_IDC` | H.264/HEVC | `arg` | `0, 1, 2` for H.264; `0, 1` for HEVC |
| `44` | `ENC_ADPTIVE_QUANTISATION` | H.264/HEVC | `arg` | Adaptive quantization control |
| `45` | `QP_DELTA_I_P` | Enc | `arg` | QP delta between I and P |
| `46` | `QP_DELTA_I_B_REF` | Enc | `arg` | QP delta between I and reference B |
| `47` | `QP_DELTA_I_B_NONREF` | Enc | `arg` | QP delta between I and non-reference B |
| `48` | `CB_QP_OFFSET` | Enc | `arg` | Cb plane QP offset |
| `49` | `CR_QP_OFFSET` | Enc | `arg` | Cr plane QP offset |
| `50` | `LAMBDA_SCALE` | Enc | `lambda_scale` | 8 `uint16_t` factors in Q8 (`0x0100` = 1.0) for I/P/B_ref/B_nonref and square roots |
| `51` | `ENC_MAX_NUM_CORES` | Enc | `arg` | Maximum number of cores for encode |
| `56` | `ENC_FIXED_QP` | Enc | `arg` | Fixed QP |
| `58` | `ENC_SAO_PENALTY` | HEVC Enc | `arg` | SAO penalty |
| `59` | `ENC_SAO_EN` | HEVC Enc | `arg` | Reserved / do not use (prefer 60 and 61) |
| `60` | `ENC_SAO_LUMA_EN` | HEVC Enc | `arg` | `0`/`1`: enables SAO on Luma |
| `61` | `ENC_SAO_CHROMA_EN` | HEVC Enc | `arg` | `0`/`1`: enables SAO on Chroma |
| `62`–`65` | `ENC_CROP_RARAM_{LEFT,RIGHT,TOP,BOTTOM}` | Enc | `arg` | *Conformance crop* parameters during encoding |
| `66` | `ENC_LTR_MODE` | Enc | `arg` | *Long Term Reference* mode |
| `67` | `ENC_LTR_PERIOD` | Enc | `arg` | LTR frame period |

---

## 3. Bitmasks and Typed Structures Breakdown (`[P]`)

### 3.1 `profile_level` (Index `3`)

| Codec | Profiles (`profile`) | Levels (`level`) |
|---|---|---|
| **H.264** | `1`=Baseline, `2`=Main, `3`=High, `4`=High 10 | `1`=1, `2`=1b, `3`=1.1, `4`=1.2, `5`=1.3, `6`=2, `7`=2.1, `8`=2.2, `9`=3, `10`=3.1, `11`=3.2, `12`=4, `13`=4.1, `14`=4.2, `15`=5, `16`=5.1, `17`=5.2, `18`=6, `19`=6.1, `20`=6.2; `32..287` = `32 + level_idc` literal |
| **HEVC** | `1`=Main, `2`=Main Still, `3`=Main Intra, `4`=Main 10 | Odd values `1,3,5,7,9,11,13,15,17,19,21,23,25` = Main Tier (`1, 2, 2.1, 3, 3.1, 4, 4.1, 5, 5.1, 5.2, 6, 6.1, 6.2`); even values `2..26` = corresponding High Tier |
| **VC-1** | `1`=Simple, `2`=Main, `3`=Advanced | — |
| **VP8** | `1`=Main | — |

### 3.2 `MVE_OPT_DISABLE_FEATURE_*` Bitmask (Index `6`)

| Bit / Mask | Macro | Disabled Feature |
|---|---|---|
| `0x0000_0001` | `AFBC_COMP` | AFBC compression in VDMA |
| `0x0000_0002` | `REF_CACHE` | REF subsystem cache |
| `0x0000_0004` | `DEBLOCK` | Deblocking filter |
| `0x0000_0008` | `SAO` | SAO filter (HEVC) |
| `0x0000_0020` | `PIC_OUTPUT` | Disables picture output writes |
| `0x0000_0040` | `PIPE` | Pipe stage (operates in *parser-only* mode) |
| `0x0000_0080` | `SLEEP` | Internal clock gating (`SYSCTRL.SLEEP`) |
| `0x0000_0100` | `AFBC_LEGACY_REF` | Enables tiled AFBC format in reference buffers (ignored for decode AFBC output) |
| `0x0000_0400` | `REF_PICS` | Forces static 16×16 references |
| `0x0000_0800` | `CHNG_RECT_WA` | Disables *change rectangles* workaround |
| `0x0000_1000` | `REFSZ_LIMIT` | Disables `REFSZ` bandwidth limit |

### 3.3 `MVE_MBTYPE_*` Bitmask (Indices `25` and `26`)

- Inter: `4x4=0x0001`, `4x8=0x0002`, `8x4=0x0004`, `8x8=0x0008`, `8x16=0x0010`, `16x8=0x0020`, `16x16=0x0040`, `PSKIP=0x0080`, `16x32=0x1000`, `32x16=0x2000`, `32x32=0x4000`.
- Intra: `I4x4=0x0100`, `I8x8=0x0200`, `I16x16=0x0400`, `I32x32=0x0800`.

### 3.4 JPEG Table Structures (Indices `32` and `33`)

```c
struct {
    uint32_t type; /* 1=DC_LUMA, 2=AC_LUMA, 3=DC_CHROMA, 4=AC_CHROMA */
    uint8_t  number_of_huffman_of_code_length[16];
    uint8_t  table[162]; /* 12 used for DC, 162 for AC */
} huffman_table;

struct {
    uint32_t type; /* 1=LUMA, 2=CHROMA */
    uint8_t  matrix[64];
} quant_table;
```

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
