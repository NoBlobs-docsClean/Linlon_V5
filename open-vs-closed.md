<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Open Sources vs. Proprietary Blobs and Cross-Generation Compatibility (`open-vs-closed`)

This document clarifies the open target contract of this specification, what belongs to the open-source ecosystem versus closed blobs, how hardware facts remain consistent across the target SoCs (**SpacemiT K1** and **SpacemiT K3**), and the differences between MVE protocol revisions (v2 vs. v3.5).

---

## 1. What Is Open and What Is Closed in the VPU Stack (`[P][O]`)

| Layer | License Status | Role in Clean-Room Reimplementation |
|---|---|---|
| **Linux `mvx` Driver** (`vpu_k1x` / `vpu_k3`) | **Open-source (`GPL-2.0-only`)** | **Normative contract `[P]`:** defines APB registers (`0xC050_0000`), LSID, MMU, `.fwb` format (`struct mvx_fw_header`), MVE 3.x protocol (`mve_protocol_def.h`), and V4L2 M2M. |
| **`ARM-software/rasc-devkit` Toolchain** | **Open-source (`GPLv2`)** | **Normative contract `[P]`:** defines the complete RASC ISA table (`rasc-opc.c`), assembler/linker (`binutils 2.18`), and C compiler (`GCC 4.2.3`) + `rasc-elf` ABI. |
| **SpacemiT K1/K3 Device Trees and Datasheets** | **Public** | **Platform contract `[P]`:** base addresses, IRQ 74, clocks, OPPs, resets, power domains, and DMA translation. |
| **`linlon-v52_v76-80-2/*.fwb` Blobs** | **Closed / Proprietary** | **Forbidden to the clean team:** this is precisely the component that this specification enables replacing from scratch in a clean room. |

---

## 2. Target Contract: MVE 3.5 Protocol (vs. Legacy MVE v2) (`[P]`)

The Linux `mvx` driver supports two major versions of the host↔firmware protocol (`protocol_major = 2` and `protocol_major = 3`):

| Aspect | MVE v2 (Legacy: Mali-V500/V550/V61) | **MVE 3.5 (Target of This Specification: Linlon V5 K1/K3)** |
|---|---|---|
| `.fwb` header | `protocol_major = 2` | **`protocol_major = 3`, `protocol_minor = 5`** |
| Virtual region map | `fw_v2/mve_protocol_def.h` | **`fw_v3/mve_protocol_def.h`** (8 FW instances, queues `0x1007_9000..F000`, `ram_print` `0x1010_0000`, `PROTECTED` `0x2000_0000`, `FRAMEBUF` `0x7000_0000`) |
| `IDLE_ACK` (`1012`) | Absent in v2 | **Active when `major == 3 && minor >= 1`** |
| H.264 High 10 (`MVE_OPT_PROFILE_H264_HIGH_10`) | Not mapped in v2 | **Supported in v3** (`to_mve_profile_v3`) |
| Arbitrary downscale (DSL) and extended options (`57..69`) | Partial/absent | **Present in the MVE 3.5 contract** |

**Normative recommendation:** The clean-room reimplementation must declare **`protocol_major = 3`, `protocol_minor = 5`** in the `.fwb` header and follow exclusively the **MVE 3.5** contract documented in this folder.

---

## 3. Hardware Continuity: SpacemiT K1 vs. SpacemiT K3 (`[P][O]`)

Empirical verification across the two target SpacemiT SoCs shows that both instantiate the exact same Linlon V5 VPU IP (`0x56635002`), differing only in core count and operating frequency:

| Platform | `HARDWARE_ID` Family | `NCORES` | MVE Protocol | RASC ISA / ABI | Implication for the Clean Team |
|---|---|---:|---|---|---|
| **SpacemiT K1** (BPI-F3) | `0x56635002` (`Linlon V5`) | 2 | **3.5** | 32-bit RASC | **Primary target** (validated live with custom RASC code) |
| **SpacemiT K3** (Pico ITX) | `0x56635002` (`Linlon V5`) | 4 | **3.5** | **Identical** | **Identical primary target:** same `.fwb` binary runs on both SoCs; 4-core topology is host-driven |

---

## 4. Known Divergences Between Silicon Revision and Driver (`[P][O]`)

1. **Silicon `HARDWARE_ID` (`0x56635002`) vs. driver enum (`0x5276`):**
   - The kernel boot log displays `hw_id=00005276` (the `MVE_v52_v76` enum value), whereas the physical register `0xC050_0000` (and internal register `0xFF02_B068`) returns **`0x56635002`** (family `0x5663`, rev `0x50` = 80, patch `0x02`).
   - The family check in firmware should mask revision/patch (e.g., `(id & 0xFFFF0000) == 0x56630000`) to accept all revisions of the Linlon V5 family (`0x5663`).
2. **Mainline vs. Vendor Kernel:**
   - The current mainline Linux kernel does not yet include the `mvx` driver or the `linlon-v5@c0500000` DT node; therefore, development and real-board validation use the `mvx` driver (GPL-2.0, buildable as the `amvx.ko` module) on a vendor kernel or ported with the DT node documented in [soc-integration.md](soc-integration.md).

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
