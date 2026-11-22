<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# MVE 3.5 Protocol — Buffer Descriptors and Pixel Formats

This document specifies the four buffer codes (`3001`–`3004`) carried on the `BUF_INQ`/`BUF_INRQ` and `BUF_OUTQ`/`BUF_OUTRQ` queues, the exact layouts of frame (Planar and AFBC), bitstream, inline parameter, and general buffer structures, the 16-bit `format` bitfield, and the validations required by the contract (`mve_protocol_def.h`, `[P][D]`).

---

## 1. Buffer Codes (`mve_msg_header.code`) (`[P]`)

| Code | Public Macro | Structure | Direction / Usage |
|---:|---|---|---|
| `3001` | `MVE_BUFFER_CODE_FRAME` | `struct mve_buffer_frame` | Output frame (decode) or input frame (encode) |
| `3002` | `MVE_BUFFER_CODE_BITSTREAM` | `struct mve_buffer_bitstream` | Input bitstream (decode) or output bitstream (encode) |
| `3003` | `MVE_BUFFER_CODE_PARAM` | `struct mve_buffer_param` | Inline parameter in the buffer queue (applies to subsequent buffers) |
| `3004` | `MVE_BUFFER_CODE_GENERAL` | `struct mve_buffer_general` | Structured auxiliary data (encode block configs / decode AD stats) |

> **Preservation of `host_handle` and `user_data_tag` (`[P]`):**
> - `host_handle` (64-bit) identifies the buffer object in the host driver and must **never** be modified by the firmware when returning the buffer.
> - `user_data_tag` (64-bit) associates input and output: the firmware **must** propagate `user_data_tag` from the bitstream buffer to the produced frame buffer(s) (decode) and vice versa (encode).

---

## 2. `struct mve_buffer_frame` (`3001`) (`[P]`)

```c
struct mve_buffer_frame_planar {
    uint32_t plane_top[3];       /* VAs of Y, Cb, Cr planes (top field or progressive frame) */
    uint32_t plane_bot[3];       /* VAs of Y, Cb, Cr planes (bottom field, interlaced only) */
    int32_t  stride[3];          /* Row stride in bytes (per plane) */
    uint16_t max_frame_width;    /* Maximum width allocated in the buffer */
    uint16_t max_frame_height;   /* Maximum height allocated in the buffer */
};                               /* sizeof = 40 bytes */

struct mve_buffer_frame_afbc {
    uint32_t plane[2];                     /* VAs: [0]=top/normal, [1]=bottom (interlace) or downscaled */
    uint32_t alloc_bytes[2];               /* Allocated size in bytes for each plane */
    uint16_t cropx;                        /* X crop in luma samples */
    uint16_t cropy;                        /* Y crop in luma samples */
    uint16_t afbc_width_in_superblocks[2]; /* Width in superblocks (16 or 32 px) */
    uint32_t afbc_params;                  /* MVE_BUFFER_FRAME_AFBC_* bitmask */
};                                         /* sizeof = 28 bytes */

struct mve_buffer_frame {
    uint64_t host_handle;          /* +0x00 */
    uint64_t user_data_tag;        /* +0x08 */
    uint32_t frame_flags;          /* +0x10 */
    uint16_t visible_frame_height; /* +0x14 */
    uint16_t visible_frame_width;  /* +0x16 */
    uint16_t format;               /* +0x18 (16-bit bitfield) */
    uint16_t reserved0;            /* +0x1A */
    union {                        /* +0x1C (40 bytes) */
        struct mve_buffer_frame_planar planar;
        struct mve_buffer_frame_afbc   afbc;
    } data;
    uint32_t reserved1;            /* +0x44 (forces sizeof = 72 bytes, multiple of 8) */
};
```

### 2.1 `frame_flags` (`MVE_BUFFER_FRAME_FLAG_*`)

| Mask | Name | Set By (Dec / Enc) | Semantics |
|---|---|---|---|
| `0x0000_0001` | `INTERLACE` | Host / — | Interlaced buffer (both top and bottom fields allocated) |
| `0x0000_0002` | `BOT_FIRST` | FW / — | Bottom field must be displayed first |
| `0x0000_0004` | `TOP_PRESENT` | FW / Host | Top field (or complete progressive frame) present |
| `0x0000_0008` | `BOT_PRESENT` | FW / — | Bottom field present |
| `0x0000_0010` | `ROTATION_90` | Host / Host | 90° rotation |
| `0x0000_0020` | `ROTATION_180` | Host / Host | 180° rotation |
| `0x0000_0030` | `ROTATION_270` | Host / Host | 270° rotation (not supported for AFBC in decode) |
| `0x0000_00C0` | `SCALING_MASK` | Host / — | Fixed downscale factor |
| `0x0000_0100` | `MIRROR_HORI` | — / Host | Horizontal mirroring (encode only; forbidden in decode) |
| `0x0000_0200` | `MIRROR_VERT` | — / Host | Vertical mirroring (encode only; forbidden in decode) |
| `0x0000_1000` | `REJECTED` | FW / — | Buffer too small; host must reallocate |
| `0x0000_2000` | `CORRUPT` | FW / — | Frame contains visual corruption |
| `0x0000_4000` | `DECODE_ONLY` | FW / — | Frame decoded but must not be displayed |
| `0x0000_8000` | `REF_FRAME` | FW / — | Frame in use as a reference (host must not modify; re-queue or use `RELEASE_REF_FRAME`) |
| `0x0001_0000` | `EOS` | FW / Host | Last frame of the stream (*End of Stream*) |
| `0x00FE_0000` | `SCALING_MASKY` | Host / — | Arbitrary vertical scale (7 bits) |
| `0xFF00_0000` | `SCALING_MASKX` | Host / — | Arbitrary horizontal scale (8 bits) |

> **Host driver note (`[P]`):** On decode return, if `visible_frame_width == 0`, `visible_frame_height == 0`, or `(frame_flags & (TOP_PRESENT | BOT_PRESENT)) == 0`, the driver zeroes `bytesused` on the V4L2 buffer planes. Therefore, every valid decoded frame **must** set `TOP_PRESENT` (and `BOT_PRESENT` if interlaced) in addition to `visible_frame_width/height`.

### 2.2 AFBC Parameters (`afbc_params`)

| Mask | Name (`MVE_BUFFER_FRAME_AFBC_*`) | Meaning |
|---|---|---|
| `0x0000_0001` | `TILED_BODY` | AFBC body blocks in tiled layout |
| `0x0000_0002` | `TILED_HEADER` | AFBC headers in tiled layout |
| `0x0000_0004` | `32X8_SUPERBLOCK` | 32×8 superblock (default = 16×16; supported only as encode input) |
| `0x0000_0008` | `DN_FORCE_8BIT` | Downscaled AFBC plane in 8-bit even if full-scale is 10-bit |
| `0x0000_0010` | `DN_FORCE_420` | Downscaled AFBC plane in 4:2:0 even if full-scale is 4:2:2 |
| `0x0000_0020` | `STRIDE_SET_BY_MVE` | Decode: firmware selects `afbc_width_in_superblocks` if this bit is set or value is 0 |
| `0x0000_0040` | `BLOCK_SPLIT` | Enables superblock `block_split` mode |

### 2.3 `format` Bitfield Encoding (16 bits)

```text
  15  14 13  12 11 10 9   8 7        4  3  2       0
+----+-----+----+----+-----+----------+---+---------+
|  A | res | V  |res |  N  |    B     |res|    C    |
+----+-----+----+----+-----+----------+---+---------+
```

- `C` (`[2:0]`): Chroma subsampling (`0=MONO`, `1=420`, `2=422`, `3=440`, `4=ARGB`).
- `B` (`[7:4]`): `bitdepth_max - 8` (`0` = 8-bit, `2` = 10-bit, `8` = 16-bit/P010/Y210).
- `N` (`[9:8]`): Number of planes (`1`, `2`, or `3`; `0` for AFBC).
- `V` (`[13:12]`): Format variant (e.g., `0=NV12`, `1=NV21`; `0..3` for RGBA/BGRA/ARGB/ABGR channel order).
- `A` (`[15]`): `1` = AFBC compressed, `0` = non-AFBC (planar/semi-planar/packed).

| Format (`MVE_FORMAT_*`) | Hex Value | Planes | Description |
|---|---:|---:|---|
| `YUV420_AFBC_8` | `0x8001` | AFBC | YUV 4:2:0 AFBC 8-bit |
| `YUV420_AFBC_10` | `0x8021` | AFBC | YUV 4:2:0 AFBC 10-bit |
| `YUV422_AFBC_8` | `0x8002` | AFBC | YUV 4:2:2 AFBC 8-bit |
| `YUV422_AFBC_10` | `0x8022` | AFBC | YUV 4:2:2 AFBC 10-bit |
| `YUV420_I420` | `0x0301` | 3 | Y, U, V planar 8-bit |
| `YUV420_I420_10` | `0x0321` | 3 | Y, U, V planar 10-bit |
| `YUV420_NV12` | `0x0201` | 2 | Y + interleaved UV 8-bit |
| `YUV420_NV21` | `0x1201` | 2 | Y + interleaved VU 8-bit |
| `YUV420_P010` | `0x0281` | 2 | Y + UV 10-bit in 16-bit words (MSB-aligned) |
| `YUV420_Y0L2` | `0x0121` | 1 | YUV 4:2:0 10-bit packed |
| `YUV420_AQB1` | `0x1121` | 1 | YUV 4:2:0 10-bit packed variant |
| `YUV422_YUY2` | `0x0102` | 1 | YUYV 4:2:2 8-bit packed |
| `YUV422_UYVY` | `0x1102` | 1 | UYVY 4:2:2 8-bit packed |
| `YUV422_Y210` | `0x0182` | 1 | YUV 4:2:2 10-bit in 16-bit |
| `RGBA_8888` | `0x0104` | 1 | Encode input only |
| `BGRA_8888` | `0x1104` | 1 | Encode input only |
| `ARGB_8888` | `0x2104` | 1 | Encode input only |
| `ABGR_8888` | `0x3104` | 1 | Encode input only |
| `MBINFO` | `0x0001` | — | Debug only |
| `UNUSED` | `0x0000` | — | Unused |

---

## 3. `struct mve_buffer_bitstream` (`3002`) (`[P]`)

```c
struct mve_buffer_bitstream {
    uint64_t host_handle;           /* +0x00 */
    uint64_t user_data_tag;         /* +0x08 */
    uint32_t bitstream_flags;       /* +0x10 */
    uint32_t bitstream_alloc_bytes; /* +0x14: total allocated capacity in bytes */
    uint32_t bitstream_offset;      /* +0x18: starting offset in bytes from buf_addr */
    uint32_t bitstream_filled_len;  /* +0x1C: number of valid bytes */
    uint32_t bitstream_buf_addr;    /* +0x20: VA in the VPU MMU */
    uint8_t  frame_type;            /* +0x24: 0=I, 1=P, 2=B, 3=b */
    uint8_t  reserved[3];           /* +0x25..0x27 (sizeof = 40 bytes) */
};
```

### `bitstream_flags` (`MVE_BUFFER_BITSTREAM_FLAG_*`)

| Mask | Name | Semantics |
|---|---|---|
| `0x0000_0001` | `EOS` | End of stream |
| `0x0000_0010` | `ENDOFFRAME` | Buffer ends on a complete frame boundary |
| `0x0000_0020` | `SYNCFRAME` | Random access / IDR / keyframe |
| `0x0000_0080` | `CODECCONFIG` | Contains configuration data (SPS/PPS/VPS) |
| `0x0000_0400` | `ENDOFSUBFRAME` | Ends on a subframe/slice boundary |

> **Observed convention when returning bitstream in decode (`[O]`):** After fully consuming `N` input bytes, the firmware returns the descriptor in `BUF_INRQ` with `bitstream_filled_len = 0` and `bitstream_offset += N`.

---

## 4. `struct mve_buffer_param` (`3003`) (`[P]`)

Sent inline in the same buffer queue; `uint32_t type` followed by the `data` union:

| `type` | Macro (`MVE_BUFFER_PARAM_TYPE_*`) | Direction | Payload |
|---:|---|---|---|
| `2` | `QP` | Enc | `int32_t qp` (H.264/HEVC `0..51`; VP8 `0..63`/`0..127`; VP9 `0..63`/`0..255`) |
| `3` | `REGIONS` | Enc | `n_regions` (up to 16) × `{ mbx_left, mbx_right, mby_top, mby_bottom, int16_t qp_delta }` |
| `5` | `DISPLAY_SIZE` | Dec output | `{ uint16_t display_width; uint16_t display_height; }` |
| `6` | `RANGE_MAP` | Dec output | `{ luma_map_enabled, luma_map_value, chroma_map_enabled, chroma_map_value }` (VC-1 AP) |
| `9` | `FRAME_RATE` | Enc | `uint32_t arg` in Q16 fps (e.g., `0x001E0000` = 30.0 fps) |
| `10` | `RATE_CONTROL` | Enc | `{ rate_control_mode (0=OFF,1=STD,2=VAR,3=CONST,4=CVAR), target_bitrate, maximum_bitrate }` |
| `12`/`13`/`14` | `QP_I` / `QP_P` / `QP_B` | Enc | `int32_t qp` per frame type (with rate control disabled) |
| `15` | `COLOUR_DESCRIPTION` | Dec/Enc | HDR/VUI metadata (`flags` bit 0=mastering display, bit 1=CLL; range 0/1/2; primaries/transfer/matrix; mastering primaries/luminance; max/avg CLL; SAR; timing) |
| `16` | `FRAME_PACKING` | Enc | Stereoscopic SEI metadata (HEVC D.3.16) |
| `17` | `FRAME_FIELD_INFO` | Dec output | `{ pic_struct, source_scan_type, duplicate_flag }` |
| `18` | `GOP_RESET` | Enc | No extra payload; forces GOP reset |
| `19` | `DPB_HELD_FRAMES` | Dec output | `uint32_t arg`: count of ready frames held in DPB for reordering |
| `20` | `CHANGE_RECTANGLES` | Enc | `n_rectangles` (0..2) × `{ x_left, x_right, y_top, y_bottom }` (applies only to the next frame) |
| `21` | `RATE_CONTROL_QP_RANGE` | Enc | `{ int32_t qp_min; int32_t qp_max; }` |
| `23` | `RATE_CONTROL_HRD_BUF_SIZE` | Enc | `uint32_t arg` (HRD buffer size) |
| `25` | `RATE_CONTROL_QP_RANGE_I` | Enc | `{ int32_t qp_min; int32_t qp_max; }` specific to I-frames |
| `26` | `SEI_USER_DATA_UNREGISTERED` | Dec/Enc | `{ flags (bit 0=valid), uuid[16], user_data[221], user_data_len }` |

---

## 5. `struct mve_buffer_general` (`3004`) (`[P]`)

Header `struct mve_buffer_general_hdr` (32 bytes): `{ uint64_t host_handle, uint64_t user_data_tag, uint32_t buffer_ptr, uint32_t buffer_size, uint16_t type, uint16_t config_size, uint32_t reserved }`, followed by `config` of size `config_size`:

- `type = 1` (`MVE_BUFFER_GENERAL_TYPE_BLOCK_CONFIGS`, encode input): `blk_cfg_type` (`0x00=NONE`, `0xFF=ROW_UNCOMP`), `{ n_cols_minus1, n_rows_minus1 }`, pointing to an array of `mve_block_param_record` (`uint16_t qp_delta` with four 4-bit deltas for a 32×32 quad + `uint8_t force`: `0=NONE`, `1=FORCE_QP`).
- `type = 2` (`MVE_BUFFER_GENERAL_TYPE_AD_STATS`, decode output): `{ uint32_t frame_averages (Y[11:0], Cb[21:12], Cr[31:22]), uint16_t thumbnail_width, uint16_t thumbnail_height, uint8_t ad_stats_flags (0=PROG, 1=INTERLACED) }`.

---

## 6. Buffer Validation Rules Required of the Firmware (`[P][D]`)

1. Reject with `MVE_ERROR_INVALID_BUFFER` (`6`) if a descriptor arrives on the wrong channel (e.g., `FRAME` in decode `BUF_INQ`) or with an invalid/unsupported `mve_buffer_general` type.
2. On output frame buffers (decode):
   - Validate that the format is supported by the hardware and `FUSE`s (e.g., AFBC/P010).
   - Validate compatibility between stream and buffer: interlaced vs. progressive, bitdepth (8-bit vs. 10-bit), and chroma subsampling (4:2:0 vs. 4:2:2) in AFBC.
   - Reject mirroring in decode and rotation on AFBC output.
   - Require valid planes to reside in the `FRAMEBUF` region and unused planes to have address `0`.
   - Require `stride` to accommodate `max_frame_width`; if the buffer is too small for the current frame, return it marked with `MVE_BUFFER_FRAME_FLAG_REJECTED`.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
