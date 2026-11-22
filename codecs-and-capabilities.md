<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Supported Codecs, Firmware Selection, and Capability Matrix

This document consolidates all video and image codecs supported by the Arm China Linlon V5 hardware (`MVE_v52_v76`, `0x56635002`) on SpacemiT K1 and SpacemiT K3, the exact `.fwb` file mapping expected by the Linux driver, the pixel format matrix, and per-codec operational constraints.

---

## 1. `.fwb` File Mapping Expected by the Driver (`[P]`)

The `mvx` driver loads an independent binary per `(codec, direction)` from `linlon-v52_v76-80-2/<codec><dec|enc>.fwb`. In total there are **16 file combinations** (11 decode blobs covering 12 standards + 5 encode blobs):

| Standard / Codec | `.fwb` File (Decode) | `.fwb` File (Encode) | Contract Note |
|---|---|---|---|
| **H.264 / AVC** | `h264dec.fwb` | `h264enc.fwb` | Profiles BP / MP / HP / High 10 (MVE 3.x) |
| **HEVC / H.265** | `hevcdec.fwb` | `hevcenc.fwb` | Main / Main 10 / Main Still / Main Intra; subject to `FUSE_DISABLE_HEVC` |
| **VP8** | `vp8dec.fwb` | `vp8enc.fwb` | Subject to `FUSE_DISABLE_VPX` |
| **VP9** | `vp9dec.fwb` | `vp9enc.fwb` | Profile 0 (8-bit) and Profile 2 (10-bit); subject to `FUSE_DISABLE_VPX` |
| **JPEG / MJPEG** | `jpegdec.fwb` | `jpegenc.fwb` | Baseline sequential up to 8192×8192 (best candidate for Milestone M4!) |
| **MPEG-2** | `mpeg2dec.fwb` | — | Main Profile (decode only) |
| **MPEG-4** | `mpeg4dec.fwb` | — | Simple / Advanced Simple Profile (decode only) |
| **H.263** | **`mpeg4dec.fwb`** | — | **No dedicated blob:** the driver maps `MVX_FORMAT_H263` to `mpeg4dec.fwb` (Profile 0) |
| **VC-1** | `vc1dec.fwb` | — | Simple / Main / Advanced Profile (supports `range_mapping_enabled`) |
| **RealVideo 8/9/10** | `rvdec.fwb` | — | Subject to `FUSE_DISABLE_REAL` |
| **AVS / AVS+** | `avsdec.fwb` | — | Decode only |
| **AVS2** | `avs2dec.fwb` | — | Decode only |

---

## 2. Capability Matrix and Performance Targets (K1 vs. K3) (`[P][O]`)

### 2.1 Decoding

| Codec | Profiles | Max Resolution | K1 (2 cores @ 819 MHz) | K3 (4 cores @ 1 GHz) |
|---|---|---|---|---|
| HEVC / H.265 | Main / Main 10 | 4096×4096 | 4K@60 | up to 4K@180 |
| H.264 / AVC | BP / MP / HP / High 10 | 4096×4096 | 4K@60 | up to 4K@180 |
| VP9 | Profile 0 / Profile 2 (10-bit) | 4096×4096 | 4K@30 | scales across 4 cores |
| VP8 | Main | 2048×2048 | 1080p@60 | scales across 4 cores |
| JPEG / MJPEG | Baseline sequential | 8192×8192 | 4K@30 | scales across 4 cores |
| MPEG-2 | Main Profile | 4096×4096 | 1080p@60 | scales across 4 cores |
| MPEG-4 / H.263 | SP / ASP / H.263 P0 | 2048×2048 | 1080p@60 | scales across 4 cores |
| VC-1 | SP / MP / AP | 2048×4096 | 1080p@60 | scales across 4 cores |
| AVS / AVS+ / AVS2 / RV8–10 | Native standards | enabled in IP | supported | supported |

### 2.2 Encoding

| Codec | Profiles | Max Resolution | K1 (2 cores @ 819 MHz) | K3 (4 cores @ 1 GHz) |
|---|---|---|---|---|
| HEVC / H.265 | Main / Main 10 | 4096×4096 | 4K@30 | up to 4K@90 |
| H.264 / AVC | BP / MP / HP / High 10 | 4096×4096 | 4K@30 | up to 4K@90 |
| VP9 | Profile 0 / Profile 2 (10-bit) | 4096×4096 | 4K@30 | scales across 4 cores |
| VP8 | Main | 2048×2048 | 1080p@60 | scales across 4 cores |
| JPEG / MJPEG | Baseline sequential | 8192×8192 | 4K@30 | scales across 4 cores |

> **ISP vs. VPU Decoupling (`[P]`):** SpacemiT K1 also includes a dedicated JPEG encoder in the camera/ISP subsystem (up to 23 MP). It is a completely separate hardware block unrelated to the VPU's `jpegenc.fwb`/`jpegdec.fwb`.

---

## 3. Supported Pixel Format Matrix (`mvx_hwreg_get_formats_v52_v76`) (`[P]`)

| Pixel Format | Input (Encode) | Output (Decode) | Constraints |
|---|:---:|:---:|---|
| `YUV420_I420` | Yes | Yes | 3 planes (Y, U, V), 8-bit |
| `YUV420_NV12` / `NV21` | Yes | Yes | 2 planes (Y + UV/VU), 8-bit |
| `YUV420_P010` | Yes | Yes | 2 planes, 10-bit in 16-bit |
| `YUV420_Y0L2` / `AQB1` | Yes | Yes | 1 plane, 10-bit packed |
| `YUV422_YUY2` / `UYVY` | Yes | Yes | 1 plane, 8-bit packed |
| `YUV422_Y210` | Yes | Yes | 1 plane, 10-bit |
| `YUV420_AFBC_8` / `10` | Yes | Yes | Disabled if `FUSE` bit 0 = 1 |
| `YUV422_AFBC_8` / `10` | Yes | Yes | Disabled if `FUSE` bit 0 = 1 |
| `RGBA/BGRA/ARGB/ABGR_8888` | **Yes** | **No** | RGB→YUV conversion via `ENC_RGB_TO_YUV_MODE` in encode only |

---

## 4. Codec-Specific Behavioral Constraints (`[P][D]`)

1. **H.264 Decode:**
   - Supports NALU packing formats via `MVE_SET_OPT_INDEX_NALU_FORMAT` (`START_CODES`, `ONE_NALU`, 1/2/4-byte length prefixes) and emulation prevention (`STREAM_ESCAPING`).
   - MBAFF streams with width above 2048 pixels are rejected.
   - Emits `ctu_size = 16` in `FRAME_ALLOC_PARAM`.
2. **HEVC Decode / Encode:**
   - `ctu_size` can be 16, 32, or 64.
   - In encode, `TILES.tile_cols` must be `0` (tile rows only), `ENC_MIN_LUMA_CB_SIZE` accepts `8` or `16`, and SAO is controlled separately for luma (`60`) and chroma (`61`).
3. **VP9 Decode / Encode:**
   - In encode, `TILES.tile_rows` and `TILES.tile_cols` must be powers of 2; probability updates follow `ENC_VP9_PROB_UPDATE` (`0=OFF`, `1=IMPLICIT`, `2=EXPLICIT`).
   - Hidden frames (*alt-ref* / `show_frame = 0`) use `MVE_BUFFER_FRAME_FLAG_DECODE_ONLY` in the DPB.
4. **JPEG Encode / Decode:**
   - Has no DPB, inter-frame references, or motion compensation — ideal as the **first real hardware acceleration target (Milestone M4)** to validate `BITSTREAM` + `TRF` + `VDMA`.
   - In encode, accepts `RESYNC_INTERVAL` (`31`), `HUFFMAN_TABLE` (`32`), and `QUANT_TABLE` (`33`).
5. **VC-1 Advanced Profile Decode:**
   - Signals `sequence_parameters.range_mapping_enabled` when active: for planar output, the firmware itself applies range mapping; for AFBC output, it sends `MVE_BUFFER_PARAM_TYPE_RANGE_MAP` (`6`) for post-decode filtering by the consumer.
6. **Arbitrary Downscaling (DSL) in Decode:**
   - Configured via `DEC_DOWNSCALE` (`57`) and `DEC_DOWNSCALE_POS_MODE` (`69`); not supported for tiled/AFBC output outside documented constraints nor for inputs with any dimension smaller than 2 pixels.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
