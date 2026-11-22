<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Host-Side Registers (APB — Base `0xC050_0000`)

This document specifies all MMIO registers exposed by the Linlon V5 VPU to the host processor (RISC-V CPU) over the APB3 bus starting at base `0xC050_0000`. All accesses are **32-bit (little-endian)**, 4-byte aligned.

---

## 1. Global Registers (`[P][O]`)

Offsets relative to base `0xC050_0000`:

| Offset | Name | Access | Function |
|---:|---|---|---|
| `0x00` | `HARDWARE_ID` | R | IP identification: family, revision, and patch |
| `0x04` | `ENABLE` | R/W | Enables (`1`) or pauses (`0`) the global job scheduler |
| `0x08` | `NCORES` | R | Number of physical VPU cores present in silicon (K1 = `2`, K3 = `4`) |
| `0x0C` | `NLSID` | R | Number of session contexts (LSIDs) supported by hardware (`4`) |
| `0x10` | `CORELSID` | R | LSID currently executing on each physical core (4 bits per core; `0xF` = free) |
| `0x14` | `JOBQUEUE` | R/W | Hardware queue with 4 8-bit slots `{ncores-1[7:4], lsid[3:0]}` |
| `0x18` | `IRQVE` | R | Global mask of pending interrupts per LSID (bit `i` = LSID `i`) |
| `0x24` | `CLKFORCE` | R/W | Forced clock control (`0` after reset to enable clock gating) |
| `0x34` | `FUSE` | R | Fuse bitmask disabling features/codecs |
| `0x38` | `CONFIG` | R | Silicon configuration/topology register |
| `0x40` | `PROTCTRL` | R/W | Global protected-session control |
| `0x50` | `RESET` | W/R | Writing `1` triggers internal VPU block reset |

---

## 2. Global Register Field Details (`[P][O]`)

### 2.1 `HARDWARE_ID` (`0x00`)

```text
 31                16 15         8 7          0
+--------------------+------------+------------+
|     FAMILY_ID      |  REVISION  |   PATCH    |
+--------------------+------------+------------+
```

| Bits | Field | Known Values |
|---|---|---|
| `[31:16]` | `FAMILY_ID` | `0x5650` = Mali-V500; `0x5655` = Mali-V550; `0x5660`/`0x5661` = Mali-V61; **`0x5662`/`0x5663` = `MVE_v52_v76` (Linlon V5/V7)** |
| `[15:8]` | `REVISION` | `0x50` (80 decimal) on K1 and K3 |
| `[7:0]` | `PATCH` | `0x02` on K1 and K3 |

- **Board-measured value (K1 and K3):** `0x56635002` (`[O]`).

### 2.2 `CORELSID` (`0x10`)

Each physical core `c` (`0 <= c < NCORES`) occupies a 4-bit nibble:

```text
core_lsid(c) = (CORELSID >> (4 * c)) & 0xF
```

- `0..3`: core `c` is executing the session mapped to the indicated LSID.
- `0xF` (`15`): core `c` is **free / idle**.
- At idle: K1 (`NCORES = 2`) reads `0x0000_00FF`; K3 (`NCORES = 4`) reads `0x0000_FFFF` (`[O]`).

### 2.3 `JOBQUEUE` (`0x14`)

Contains **4 8-bit slots** (`nr = 0..3`, LSB to MSB):

```text
 31       24 23       16 15        8 7         0
+-----------+-----------+-----------+-----------+
|  SLOT 3   |  SLOT 2   |  SLOT 1   |  SLOT 0   |
+-----------+-----------+-----------+-----------+

Each SLOT (8 bits):
  7       4 3       0
 +---------+---------+
 | NCORES-1|  LSID   |
 +---------+---------+
```

| Slot Field | Bits | Constant / Rule |
|---|---|---|
| `LSID` | `[3:0]` | `0..3` = queued LSID; **`0xF`** (`MVE_JOBQUEUE_JOB_INVALID`) = empty/invalid slot |
| `NCORES-1` | `[7:4]` | Number of cores requested for the job minus 1 (`0` = 1 core, `1` = 2 cores, `3` = 4 cores) |

- At idle (empty queue): `0x0F0F_0F0F` (`[O]`).
- **Modification protocol (`[P]`):** Before read-modify-writing `JOBQUEUE`, the host **must** disable the scheduler (`ENABLE = 0`), update `JOBQUEUE`, and re-enable (`ENABLE = 1`).

### 2.4 `IRQVE` (`0x18`)

- Bit `i` (`1 << i`, for `0 <= i < NLSID`) indicates that LSID `i` has an active interrupt directed to the host (triggered by firmware via internal register `0xFF02B028`).
- Interrupt clearing is performed in the per-LSID `LIRQVE` register (by writing `0`), not in `IRQVE`.

### 2.5 `FUSE` (`0x34`)

Each bit set to `1` **disables** the corresponding hardware feature:

| Bit | Mask | Public Macro | Effect when `1` |
|---:|---|---|---|
| 0 | `0x0000_0001` | `MVX_HWREG_FUSE_DISABLE_AFBC` | Disables AFBC formats (8-bit and 10-bit, 4:2:0 and 4:2:2) |
| 1 | `0x0000_0002` | `MVX_HWREG_FUSE_DISABLE_REAL` | Disables RealVideo (`RV`) decoding |
| 2 | `0x0000_0004` | `MVX_HWREG_FUSE_DISABLE_VPX` | Disables VP8 and VP9 codecs |
| 3 | `0x0000_0008` | `MVX_HWREG_FUSE_DISABLE_HEVC` | Disables HEVC (H.265) codec |

- **Board-measured value (K1 and K3):** `0x0000_0000` (no features disabled) (`[O]`).

### 2.6 `CONFIG` (`0x38`) and `PROTCTRL` (`0x40`)

- Individual bitfields of `CONFIG` and `PROTCTRL` have no public macros in the Linux driver (`[P]` defines no fields; measured values `[O]`):
  - `CONFIG`: `0xFFFF6402` on K1 (2 cores) and `0xFFFF7404` on K3 (4 cores) — XOR difference `0x00001006` tracks core topology.
  - `PROTCTRL`: `0x00000001` at idle on K1 and K3.

---

## 3. Per-LSID Registers (`[P][O]`)

Each session context `lsid` (`0 <= lsid < NLSID`, i.e., `0..3`) occupies a `0x40`-byte block starting at:

```text
LSID_BASE(lsid) = 0xC050_0000 + 0x0200 + 0x40 * lsid
```

| LSID | Base Address |
|---:|---|
| 0 | `0xC050_0200` |
| 1 | `0xC050_0240` |
| 2 | `0xC050_0280` |
| 3 | `0xC050_02C0` |

### 3.1 Per-LSID Offset Map

| Offset | Name | Access | Function and Fields |
|---:|---|---|---|
| `+0x00` | `CTRL` | R/W | `[7:0]` `DISALLOW` (bitmask of disallowed cores: bit `c = 1` disallows core `c`); `[11:8]` `MAXCORES` (maximum cores allowed for the LSID) |
| `+0x04` | `MMU_CTRL` | R/W | Root PTE (L0/L1) of the session page table (same 32-bit format as MMU PTEs, pointing to the L1 physical page via `phys_cpu2vpu`) |
| `+0x08` | `NPROT` | R/W | Session protection/protected-region configuration (saved/restored in PCB; observed idle value = `1`) |
| `+0x0C` | `ALLOC` | R/W | LSID allocation state: `0` = `MVE_ALLOC_FREE`, `1` = `MVE_ALLOC_NON_PROTECTED`, `2` = `MVE_ALLOC_PROTECTED` |
| `+0x10` | `FLUSH_ALL` | W | Writing `0` invalidates all TLB/MMU entries for this LSID |
| `+0x14` | `SCHED` | R/W | `1` enables scheduling of this LSID; `0` disables |
| `+0x18` | `TERMINATE` | R/W | Writing `1` requests immediate LSID termination; hardware clears the register (`0`) when termination completes (mandatory poll until `0`) |
| `+0x1C` | `LIRQVE` | R/W | Local VPU → Host IRQ status; writing `0` (+ `wmb()` barrier) clears this LSID's interrupt |
| `+0x20` | `IRQHOST` | R/W | Writing `1` signals a Host → Firmware interrupt for this LSID (also saved/restored in PCB across unmap/remap) |
| `+0x24` | `INTSIG` | R/W | Internal signaling register (no public bit documentation; reads `0` at idle) |
| `+0x2C` | `STREAMID` | R/W | Stream identifier for SMMU/bus (reads `0` at idle) |
| `+0x30` | `BUSATTR_0` | R/W | AXI attributes for pages with `MVX_ATTR_PRIVATE` (`AT = 0`); driver default: `0x00` (DT) or `0x33` observed |
| `+0x34` | `BUSATTR_1` | R/W | AXI attributes for pages with `MVX_ATTR_REFFRAME` (`AT = 1`); driver default: `0x00` (DT) or `0x33` observed |
| `+0x38` | `BUSATTR_2` | R/W | AXI attributes for pages with `MVX_ATTR_SHARED_RO` (`AT = 2`); driver default: `0x33` |
| `+0x3C` | `BUSATTR_3` | R/W | AXI attributes for pages with `MVX_ATTR_SHARED_RW` (`AT = 3`); driver default: `0x33` |

### 3.2 `CTRL` Register Detail (`+0x00`)

```text
 31             12 11       8 7              0
+-----------------+----------+----------------+
|    Reserved     | MAXCORES |    DISALLOW    |
+-----------------+----------+----------------+
```

- `MVE_CTRL_DISALLOW_SHIFT = 0`, `MVE_CTRL_DISALLOW_MASK = 0xFF`:
  - For `ncores` physical cores, bits for non-existent cores are marked disallowed: `disallow = (0xFFFFFFFF << ncores) & 0xFF`.
  - K1 example (`ncores = 2`): `disallow = 0xFC` (allows cores 0 and 1).
  - K3 example (`ncores = 4`): `disallow = 0xF0` (allows cores 0, 1, 2, and 3).
- `MVE_CTRL_MAXCORES_SHIFT = 8`, `MVE_CTRL_MAXCORES_MASK = 0xF`:
  - Configured with the maximum number of cores the session may use (`1..ncores`).

---

## 4. Snapshots Measured on Real Hardware (Idle State) (`[O]`)

### 4.1 Global Registers

| Register | SpacemiT K1 (2 cores) | SpacemiT K3 (4 cores) |
|---|---|---|
| `HARDWARE_ID` | `0x56635002` | `0x56635002` |
| `ENABLE` | `0x00000000` | `0x00000000` |
| `NCORES` | `0x00000002` | `0x00000004` |
| `NLSID` | `0x00000004` | `0x00000004` |
| `CORELSID` | `0x000000FF` | `0x0000FFFF` |
| `JOBQUEUE` | `0x0F0F0F0F` | `0x0F0F0F0F` |
| `IRQVE` | `0x00000000` | `0x00000000` |
| `CLKFORCE` | `0x00000000` | `0x00000000` |
| `FUSE` | `0x00000000` | `0x00000000` |
| `CONFIG` | `0xFFFF6402` | `0xFFFF7404` |
| `PROTCTRL` | `0x00000001` | `0x00000001` |
| `RESET` | `0x00000000` | `0x00000000` |

### 4.2 Per-LSID Registers (`lsid0..3` at idle, K1 and K3)

```text
CTRL       = 0x00000000    SCHED      = 0x00000000    BUSATTR_0  = 0x00000033
MMU_CTRL   = 0x00000000    TERMINATE  = 0x00000000    BUSATTR_1  = 0x00000033
NPROT      = 0x00000001    LIRQVE     = 0x00000000    BUSATTR_2  = 0x00000033
ALLOC      = 0x00000000    IRQHOST    = 0x00000000    BUSATTR_3  = 0x00000033
FLUSH_ALL  = 0x00000000    INTSIG     = 0x00000000    STREAMID   = 0x00000000
```

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
