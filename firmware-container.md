<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Firmware Container Format (`.fwb`) and Loading Contract

This document specifies the `.fwb` binary format expected by the Linux `mvx` driver, the validations performed during loading, the MMU mapping, and the mandatory rules for a clean-room compiled firmware image to boot correctly on real SpacemiT K1/K3 hardware.

---

## 1. `.fwb` File Path Resolution (`[P]`)

The driver constructs the relative path under `/lib/firmware/` (or `/usr/lib/firmware/`) from the values read in `HARDWARE_ID`:

```text
linlon-<hw_id_str>-<revision>-<patch>/<codec><dir>.fwb
```

For SpacemiT K1 and SpacemiT K3 (`HARDWARE_ID = 0x56635002`, `MVE_v52_v76`, `revision = 80`, `patch = 2`):

```text
linlon-v52_v76-80-2/<codec><dec|enc>.fwb
```

Examples: `linlon-v52_v76-80-2/h264dec.fwb`, `linlon-v52_v76-80-2/jpegdec.fwb`, `linlon-v52_v76-80-2/hevcenc.fwb`. The complete list of codec mappings (including H.263 → `mpeg4dec.fwb`) is in [codecs-and-capabilities.md](codecs-and-capabilities.md).

---

## 2. Header Structure (`struct mvx_fw_header`, 180 Bytes) (`[P]`)

The `.fwb` file is a **flat binary image** with no ELF header, no relocations, and no compression. Its first **180 bytes (`0x00`–`0xB3`)** form `struct mvx_fw_header` (all integers are **little-endian**):

| Offset | Size | Type | Field | Contract / Description |
|---:|---:|---|---|---|
| `0x00` | 4 | `uint32_t` | `rasc_jmp` | **Executable reset vector.** These are the first 4 bytes executed by the RASC CPU at `VA 0x1000` (must contain a `br` branch past the header; see §4). |
| `0x04` | 1 | `uint8_t` | `protocol_minor` | MVE protocol minor version. Must be **`5`** for MVE 3.5 (with `major=3`, `minor>=1` enables `IDLE_ACK`). |
| `0x05` | 1 | `uint8_t` | `protocol_major` | MVE protocol major version. Must be **`3`** (driver accepts `2` or `3`; `3` enables the MVE 3.x map and H.264 High 10). |
| `0x06` | 2 | `uint8_t[2]` | `reserved` | Reserved (`0` in the public contract). |
| `0x08` | 56 | `uint8_t[56]` | `info_string` | Null-terminated human-readable ASCII string (`\0`) describing target and codec (printed in the driver log). |
| `0x40` | 8 | `uint8_t[8]` | `part_number` | ASCII identifier up to 8 bytes (e.g., `"H264DEC"`). |
| `0x48` | 8 | `uint8_t[8]` | `svn_revision` | ASCII build revision identifier (8 bytes). |
| `0x50` | 16 | `uint8_t[16]` | `version_string` | ASCII firmware version string (up to 16 bytes). |
| `0x60` | 4 | `uint32_t` | `text_length` | **Total length in bytes to copy starting from file offset `0x00`** (that is, **it includes the 180-byte header + padding + all code/rodata**). |
| `0x64` | 4 | `uint32_t` | `bss_start_address` | Starting virtual address of the BSS segment (relative to `core_base`, **4 KiB-aligned** and strictly after the end of the `text` segment). |
| `0x68` | 4 | `uint32_t` | `bss_bitmap_size` | Number of 4 KiB pages described by `bss_bitmap` (maximum `16 * 32 = 512` pages = 2 MiB; within the 1 MiB instance window the practical limit is smaller). |
| `0x6C` | 64 | `uint32_t[16]` | `bss_bitmap` | Little-endian bitmap (bit `i` = `(bss_bitmap[i >> 5] >> (i & 31)) & 1`). If `1` (and outside `master_rw`), allocates a **per-core private** BSS page at `core_base + bss_start_address + i * 0x1000`. |
| `0xAC` | 4 | `uint32_t` | `master_rw_start_address` | Starting virtual address (relative to `core_base`, 4 KiB-aligned) of the **shared** BSS region across all cores in the session. |
| `0xB0` | 4 | `uint32_t` | `master_rw_size` | Size in bytes of the shared `master_rw` region (multiple of 4 KiB; `0` if unused). |

---

## 3. Exact Driver Validation and Loading Algorithm (`[P]`)

When loading a `.fwb` in non-secure mode (`mvx_firmware_cache.c` and `mvx_firmware.c`):

1. **Validation:**
   - Rejects (`-EFAULT`) if `fw->size < 180` (`sizeof(struct mvx_fw_header)`).
   - Rejects (`-EFAULT`) if `header->text_length > fw->size`.
   - Rejects (`-EINVAL`) if `protocol_major` is neither `2` nor `3`.
2. **Page counting:**
   - `text_cnt = ceil(header->text_length / 4096)`.
   - Iterates `i = 0 .. header->bss_bitmap_size - 1` with `va = header->bss_start_address + i * 4096`:
     - If `va >= master_rw_start_address && va < master_rw_start_address + master_rw_size`: increments `sbss_cnt` (note that **this check takes precedence over the `bss_bitmap` bit**).
     - Else if `bit(i, bss_bitmap) == 1`: increments `bss_cnt`.
     - Else: page remains unmapped (guard page).
3. **Physical memory allocation (all pages zeroed):**
   - `fw->text`: `text_cnt` pages (single copy, shared RO/X across all cores).
   - `fw->bss`: `bss_cnt * ncores` pages (private per core).
   - `fw->bss_shared`: `sbss_cnt` pages (single copy, shared RW across all cores).
4. **Per-core MMU mapping (`core = 0 .. ncores - 1`, `fw_base = core * 0x0100_0000`):**
   - Maps `fw->text` at `fw_base + 0x1000` with `MVX_ATTR_PRIVATE`, `MVX_ACCESS_EXECUTABLE`.
   - Maps each `master_rw` page at `fw_base + va` pointing to `fw->bss_shared` with `MVX_ATTR_PRIVATE`, `MVX_ACCESS_READ_WRITE`.
   - Maps each private page with bit `1` at `fw_base + va` pointing to `fw->bss[core * bss_cnt + k]` with `MVX_ATTR_PRIVATE`, `MVX_ACCESS_READ_WRITE`.
5. **Image copy:**
   - Copies `header->text_length` bytes starting from **offset `0x00` of the `.fwb` file** to `VA 0x1000` (since the physical pages of `fw->text` are shared, the copy is immediately visible at `0x0000_1000`, `0x0100_1000`, ..., `(ncores-1)*0x0100_0000 + 0x1000`).
   - Executes `dma_sync_single_for_device` to ensure DRAM coherency.

---

## 4. Mandatory Rules for Building a Clean-Room `.fwb` (`[P][O]`)

Experimental execution on real K1 hardware confirmed four indispensable rules; violating any of them prevents the firmware from running or causes silent failure:

### Rule 1 — `text_length` MUST include the 180-byte header (`[P][O]`)
Because the driver copies `text_length` bytes starting from byte `0` of the file, setting `text_length` to only the code size after the header will truncate the last 180 bytes of the image in memory. Always set `text_length = total_file_size` (or `code_offset + code_and_rodata_size`).

### Rule 2 — File offset `0x00` MUST contain a `br` trampoline (`[O]`)
The hardware starts execution of CPU RASC0 at the first byte of the `text` segment (`VA 0x1000` = file offset `0x00`), where `struct mvx_fw_header` itself resides. Therefore, the 4 bytes of `rasc_jmp` at offset `0x00` must be a valid RASC unconditional branch instruction (`br`) jumping to the entry point past the header (`offset >= 0xB4`, e.g., `0xB4` or `0xC0`, with suitable alignment):
- Example RASC encoding of `br` with a 2-byte suffix (3-byte instruction + 1 padding byte `0x00`):
  - Byte `0`: `0x5E` (`br`, 2-bit range field = `1` → signed 2-byte suffix).
  - Bytes `1..2`: `delta = (target_offset - 1)` as 16-bit little-endian (`target = pc + 1 + delta`, since `base_len = 1` for `br`).
  - Byte `3`: `0x00` (padding).
  - Note: to jump from offset `0x00` to offset `0xB4`, `delta = 0xB4 - 1 = 0x00B3` → bytes `5E B3 00 00`. To jump to offset `0xC0`, `delta = 0xC0 - 1 = 0x00BF` → bytes `5E BF 00 00`.

### Rule 3 — Alignment Between Linker Script and Payload Offset (`[O]`)
Every code and data address in the linker script must match the fixed relationship:
```text
VA_in_instance_0 = 0x1000 + file_offset
```
If the linker generates base addresses starting at `0x0000` (file offset) and uses exclusively PC-relative instructions (`br`, `bsr`, `lpc`), the image works at `0x0000_1000` as well as `0x0100_1000`, `0x0200_1000`, `0x0300_1000` without needing relocation. If absolute pointer tables exist in `.rodata`, account for the `0x1000` offset (and beware of `core >= 1` instances, whose base differs by `core * 0x0100_0000`).

### Rule 4 — PC-Relative Stack Pointer (`sp`) Initialization (`[O]`)
Each RASC CPU's stack resides in the instance's private BSS. Loading `sp` with a fixed absolute constant (e.g., `mov sp, #0x80000`) fails on any `core >= 1` instance (where private BSS is shifted by `core * 0x0100_0000`). Always initialize `sp` relative to PC:
```asm
lpc sp, stack_top
```

---

## 5. Recommended Layout of a Clean-Room `.fwb` Image (`[D]`)

```text
File offset          VA (Core c, base = c * 0x0100_0000)    Contents
───────────────────  ───────────────────────────────────    ──────────────────────────────
(unmapped)           base + 0x0000 .. base + 0x0FFF         Guard page (AP = 0)
0x0000 .. 0x0003     base + 0x1000 .. base + 0x1003         Trampoline `br _start` (rasc_jmp field)
0x0004 .. 0x00B3     base + 0x1004 .. base + 0x10B3         Remainder of struct mvx_fw_header
0x00B4 .. (aligned)  base + 0x10B4 ..                       Optional padding up to _start
_start .. text_end   base + 0x1000 + _start ..              .text + .rodata (executable/RO)
(outside file)       base + bss_start_address ..            Per-core private .bss + stacks (RW, zeroed)
(outside file)       base + master_rw_start_address ..      Cross-core shared master_rw (RW, zeroed)
```

- Remember to ensure that `bss_start_address` (`0x1000`-aligned) is `>= 0x1000 + align_up(text_length, 0x1000)` so it does not collide with the `text` segment mapping!
- The driver zeroes all `bss` and `bss_shared` pages upon allocation; if the firmware needs non-zero initialized global variables (`.data`), the startup code must copy them from `.rodata` (in the `text` segment) into `.bss` before entering `main()`.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
