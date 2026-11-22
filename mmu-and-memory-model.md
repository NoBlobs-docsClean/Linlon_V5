<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# VPU MMU and Virtual Memory Model (MVE 3.x)

This document specifies the architecture of the Linlon V5 VPU's 2-level MMU, the exact layout of page table entries (PTEs), the complete map of the 32-bit virtual address space defined by the MVE 3.x protocol, and the coherency rules between the host CPU and the VPU.

---

## 1. 2-Level MMU Architecture (`[P]`)

The VPU translates **32-bit** virtual addresses (`mvx_mmu_va`) into up to **40-bit** physical addresses on the AXI bus (restricted to 34 bits by `DMA_BIT_MASK(34)` on K1), using **4 KiB** pages (`MVE_PAGE_SIZE = 4096`, `MVE_PAGE_SHIFT = 12`).

- **Page walk levels:** `2` (`L1` and `L2`).
- **Entries per table page:** `1024` (`4096 / 4 bytes`, `MVE_INDEX_SHIFT = 10`).
- **Isolation:** Each LSID has its own `L1` table, whose root PTE is written to `LSID.MMU_CTRL` (`+0x04`).

### 1.1 Virtual Address Breakdown (32-bit)

```text
 31                22 21                12 11                     0
+--------------------+--------------------+------------------------+
|   L1 Index (10b)   |   L2 Index (10b)   |   Page Offset (12b)    |
+--------------------+--------------------+------------------------+
      0..1023              0..1023               0..4095
```

- `L1_index = (va >> 22) & 0x3FF` (each L1 entry covers `4 MiB`).
- `L2_index = (va >> 12) & 0x3FF` (each L2 entry covers `4 KiB`).
- `page_offset = va & 0xFFF`.

### 1.2 32-Bit PTE Format (`mvx_mmu_pte`)

The same 32-bit format is used in `MMU_CTRL`, in `L1` table entries (pointing to `L2` pages), and in `L2` table entries (pointing to code/data pages):

```text
 31 30 29                                                2 1  0
+-----+---------------------------------------------------+----+
| AT  |                  PA [39:12] (28 bits)             | AP |
+-----+---------------------------------------------------+----+
```

| Field | Bits | Shift / Mask | Meaning |
|---|---|---|---|
| `AT` | `[31:30]` | `shift = 30`, `mask = 0x3` | Memory attribute / selects which `BUSATTR_<AT>` (`0..3`) register defines the transaction's AXI attributes |
| `PA` | `[29:2]` | `shift = 2`, `mask = 0x0FFF_FFFF` | Physical page frame number in VPU bus space (`pa_vpu >> 12`, covering `PA[39:12]`) |
| `AP` | `[1:0]` | `shift = 0`, `mask = 0x3` | Access permission (`0` = invalid/no access → triggers `MMU ABORT`) |

**Normative PTE construction:**
```c
pte = ((attr & 0x3u) << 30) | (((pa_vpu >> 12) & 0x0FFFFFFFu) << 2) | (access & 0x3u);
```

### 1.3 `AT` (`enum mvx_mmu_attr`) and `AP` (`enum mvx_mmu_access`) Values

| `AT` | Name | Typical Usage | Associated `BUSATTR` |
|---:|---|---|---|
| `0` | `MVX_ATTR_PRIVATE` | L1/L2 tables, firmware executable text, private BSS, and `master_rw` | `BUSATTR_0` |
| `1` | `MVX_ATTR_REFFRAME` | Reference frames / AFBC buffers | `BUSATTR_1` |
| `2` | `MVX_ATTR_SHARED_RO` | Read-only shared memory | `BUSATTR_2` |
| `3` | `MVX_ATTR_SHARED_RW` | MVE communication queues, RPC area, `ram_print`, and shared buffers | `BUSATTR_3` |

| `AP` | Name | Permission |
|---:|---|---|
| `0` | `MVX_ACCESS_NO` | No access (invalid entry; triggers `MMU ABORT`) |
| `1` | `MVX_ACCESS_READ_ONLY` | Read-only (also used in L1 entries pointing to L2 tables) |
| `2` | `MVX_ACCESS_EXECUTABLE` | Executable / instruction fetch (used for the firmware `text` segment) |
| `3` | `MVX_ACCESS_READ_WRITE` | Read and write (used for BSS, queues, RPC, and framebuffers) |

### 1.4 TLB Invalidation

- Whenever the host maps/resizes/frees pages in an active session (e.g., `RPC MEM_RESIZE`, `RPC MEM_FREE`, or mapping new buffers), it must write `0` to `LSID.FLUSH_ALL` (`+0x10`).
- When updating any L1 or L2 entry, the host must synchronize the CPU cache to the device (`dma_sync_single_for_device` with `phys_cpu2vpu`).

---

## 2. MVE 3.x Virtual Memory Map (`[P][O]`)

The 32-bit virtual address space (`0x0000_0000` – `0xFFFF_FFFF`) is fixedly partitioned by the `mve_protocol_def.h` contract:

| Region | Start (VA) | End (exclusive) | Size | Attribute / Access | Function |
|---|---|---|---:|---|---|
| **FW Instance 0** | `0x0000_0000` | `0x0010_0000` | 1 MiB | `PRIVATE` / `X` (`text`), `RW` (`bss`/`master_rw`) | Physical core 0 (code at `+0x1000`) |
| **FW Instance 1** | `0x0100_0000` | `0x0110_0000` | 1 MiB | `PRIVATE` / `X`, `RW` | Physical core 1 (code at `0x0100_1000`) |
| **FW Instance 2** | `0x0200_0000` | `0x0210_0000` | 1 MiB | `PRIVATE` / `X`, `RW` | Physical core 2 (K3; code at `0x0200_1000`) |
| **FW Instance 3** | `0x0300_0000` | `0x0310_0000` | 1 MiB | `PRIVATE` / `X`, `RW` | Physical core 3 (K3; code at `0x0300_1000`) |
| **FW Instances 4–7** | `0x0400_0000` – `0x0700_0000` | `+0x0010_0000` each | 1 MiB × 4 | `PRIVATE` / `X`, `RW` | Scalability up to 8 cores |
| **MSG_INQ (Host → FW)** | `0x1007_9000` | `0x1007_A000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_host` (requests) |
| **MSG_OUTQ (FW → Host)** | `0x1007_A000` | `0x1007_B000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_mve` (responses) |
| **BUF_INQ (Host → FW)** | `0x1007_B000` | `0x1007_C000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_host` (input buffers) |
| **BUF_INRQ (FW → Host)** | `0x1007_C000` | `0x1007_D000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_mve` (returned input buffers) |
| **BUF_OUTQ (Host → FW)** | `0x1007_D000` | `0x1007_E000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_host` (output buffers) |
| **BUF_OUTRQ (FW → Host)** | `0x1007_E000` | `0x1007_F000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_mve` (returned output buffers) |
| **RPC** | `0x1007_F000` | `0x1008_0000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_rpc_communication_area` |
| **FW Print RAM** | `0x1010_0000` | `0x1018_0000` | 512 KiB | `SHARED_RW` / `RW` | Circular buffer for firmware text logs |
| **PROTECTED** | `0x2000_0000` | `0x7000_0000` | 1.25 GiB | `PRIVATE` or `SHARED_RW` / `RW` | Bitstreams, intermediate buffers (`IntBuf`), RPC `region=0` allocations |
| **FRAMEBUF** | `0x7000_0000` | `0xF000_0000` | 2.0 GiB | `REFFRAME` or `SHARED_RW` / `RW` | Planar/AFBC framebuffers, reference banks, RPC `region=1` allocations |
| **MVE Internal Use** | `0xF000_0000` | `0x1_0000_0000` | 256 MiB | Internal (not translated via DRAM) | RASC CPU internal MMIO and internal SRAMs (`0xFF00_0000`–`0xFF2F_8FFF`) |

> **Watch out for page zero of each instance (`[P]`):** Address `core_base + 0x0000` (`0x0000_0000`, `0x0100_0000`, ...) is **unmapped** (`AP = 0`), acting as a guard page against `NULL` pointer dereferences. Executable firmware code begins at **`core_base + 0x1000`** (`FW_TEXT_BASE_ADDR = 0x1000`).

---

## 3. Memory Region Rules Required by the Contract (`[P][D]`)

The firmware must validate virtual addresses received from the host and returned by RPC against the ranges above:

1. **RPC Allocations (`mve_rpc_params.mem_alloc`):**
   - `region = 0` (`MVE_MEM_REGION_PROTECTED`): the VA returned by the host **must** fall within `[0x2000_0000, 0x7000_0000)` and be a multiple of `2^(log2_alignment)`.
   - `region = 1` (`MVE_MEM_REGION_OUTBUF` / `FRAMEBUF`): the VA returned by the host **must** fall within `[0x7000_0000, 0xF000_0000)` and be a multiple of `2^(log2_alignment)`.
2. **Frame Buffers (`mve_buffer_frame`):**
   - Valid planes (`plane_top[]`, `plane_bot[]`, `afbc.plane[]`) must reside in the `FRAMEBUF` region (`[0x7000_0000, 0xF000_0000)`) — or in `PROTECTED` for secure sessions. Unused planes must have address `0`.
3. **Bitstream Buffers (`mve_buffer_bitstream`):**
   - `bitstream_buf_addr` typically resides in the `PROTECTED` region (`[0x2000_0000, 0x7000_0000)`, observed at `0x2000_0000` on K1 `[O]`).
4. **BSS pages not set in the bitmap:**
   - Any page within the BSS bitmap range whose bit is `0` (and outside `master_rw`) remains **unmapped** in the MMU; any access triggers `MMU ABORT`.

---

## 4. Multi-Core View of Firmware Instances (`[P][O]`)

When a session uses multiple cores (e.g., `ncores = 2` on K1 or `4` on K3):

- **`text` segment:** The **same physical pages** are mapped simultaneously at `0x0000_1000`, `0x0100_1000`, ..., `(ncores-1)*0x0100_0000 + 0x1000` with `MVX_ACCESS_EXECUTABLE` permission.
- **Private `bss` segment:** Each core `c` receives its own exclusive set of zeroed physical pages at `c * 0x0100_0000 + va`.
- **Shared `master_rw` segment:** The **same physical pages** are mapped at `c * 0x0100_0000 + va` for all cores `c = 0 .. ncores-1` with `MVX_ACCESS_READ_WRITE` permission.
- **MVE queues and RPC (`0x1007_9000`–`0x1007_F000`):** Unique absolute addresses shared between the host and all cores of the session.

**Mandatory consequence for firmware (`[O]`):** Because core `c` executes at `c * 0x0100_0000 + 0x1000`, all accesses to global variables in private BSS, stacks, and local tables **must** use **PC-relative (`lpc`)** addressing, and never fixed absolute constants targeting instance `0`. Fixed absolute addresses must only be used for the global regions (`0x1007_9000`–`0x1018_0000`, `PROTECTED`, `FRAMEBUF`, and internal MMIO `0xFF00_0000+`).

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
