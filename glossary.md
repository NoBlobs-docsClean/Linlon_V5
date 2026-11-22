<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Glossary — SpacemiT K1 / K3 VPU (Arm China Linlon V5)

| Acronym / Term | Definition |
|---|---|
| **AFBC** | *Arm Frame Buffer Compression* — lossless framebuffer compression format in superblocks (16×16 or 32×8) used by VDMA and the REF subsystem to reduce DDR bandwidth. |
| **APB3** | *AMBA Advanced Peripheral Bus 3* — control bus used by the SoC's RISC-V CPU to access global and per-LSID registers at `0xC050_0000`. |
| **AXI** | *AMBA Advanced eXtensible Interface* — 128-bit data bus used by the VPU to read/write DRAM (bitstreams, framebuffers, `IntBuf`, and MMU tables). |
| **BCC3** | Family of RASC conditional branch instructions with a 3-bit range field whose unsuffixed value counts variable-length **instructions** to skip, rather than bytes. |
| **BITSTREAM** | Internal entropy/bitstream accelerator block (CABAC, CAVLC, VP8/VP9 boolean decoding, and MV prediction). |
| **BSS (`bss_bitmap`)** | RW memory segment zeroed by the driver upon loading `.fwb`; private pages are allocated per core instance according to the header bitmap. |
| **Clean-Room** | Clean-room methodology where the implementation team works exclusively from behavioral specifications and public contracts, with no contact with proprietary binaries or code. |
| **CR (`CR0`–`CR255`)** | *Control Registers* of the RASC CPUs, addressed by an 8-bit index via `mtcr`/`mfcr` instructions. |
| **CTU** | *Coding Tree Unit* — basic coding unit in HEVC (16×16, 32×32, or 64×64). |
| **DBL** | *Deblocking Filter* — internal in-loop filtering block (deblocking and SAO). |
| **DPB** | *Decoded Picture Buffer* — logical pool of decoded frames retained for reference and display reordering. |
| **DSL** | *Downscale* — flexible resizing at the decoder output (`DEC_DOWNSCALE`). |
| **DVFS** | *Dynamic Voltage and Frequency Scaling* — dynamic frequency adjustment of `vpu_clk` (307.2–819.2 MHz on K1; up to 1 GHz on K3). |
| **FAR** | *Fault Address Register* — faulting virtual address captured in `CR2` during a memory exception / `MMU ABORT`. |
| **`.fwb`** | *Firmware Binary* — flat container (180-byte `mvx_fw_header` + code/rodata image) loaded by the `mvx` driver. |
| **IntBuf** | *Intermediate Buffer* — intermediate DDR buffers (allocated via RPC) that decouple the *parse* stage (`mbinfo`/residuals) from the *pipe* stage (`MC`/`TRF`/`DBL`). |
| **JOBQUEUE** | Global register `0x14` with four 8-bit slots `{ncores-1[7:4], lsid[3:0]}` used by the host to submit sessions to the hardware scheduler. |
| **Linlon V5** | Arm China's commercial brand for the VPU generation derived from the Mali-V52/V76 family (`HARDWARE_ID = 0x56635002`, `compatible = "arm china,linlon-v5"`). |
| **LSID** | *Local Session ID* — independent hardware session context (4 available: LSID 0–3), with its own MMU, IRQs, and core control. |
| **LTR** | *Long Term Reference* — long-term reference frame in H.264/HEVC encoding. |
| **`master_rw`** | RW page region declared in the `.fwb` header whose physical pages are **shared** at the same virtual offset across all cores of a multi-core session. |
| **MC / ME** | *Motion Compensation* / *Motion Estimation* — internal motion compensation and motion search blocks. |
| **MMU** | *Memory Management Unit* — 2-level VPU translation unit (32-bit VA → 40-bit PA, 4 KiB pages, 32-bit PTE). |
| **MVE 3.5** | *Mali/Linlon Video Engine Protocol v3.5* — circular queue, message, buffer, and RPC protocol between the Linux driver and firmware. |
| **`mvx`** | Arm China / SpacemiT GPL-2.0 Linux V4L2 Mem2Mem driver for the Linlon/Mali-V family. |
| **PCB** | *Process Control Block* (`struct mvx_lsid_pcb`) — set of LSID registers (`ctrl`, `mmu_ctrl`, `irqhost`, `nprot`) saved by the host when unmapping an idle session. |
| **`ram_print`** | 512 KiB region at `0x1010_0000` used as a circular text log from firmware to host (magic `0x11223356`). |
| **RASC** | 32-bit RISC microprocessor (5-stage pipeline, variable-length instructions) designed by ARM for controlling Mali-V/Linlon VPUs; there are 5 per core. |
| **REF** | Internal reference-frame management and caching subsystem (typically in AFBC). |
| **RPC** | *Remote Procedure Call* — synchronous channel at `0x1007_F000` used by firmware to request `DEBUG_PRINTF`, `MEM_ALLOC`, `MEM_RESIZE`, and `MEM_FREE` from the host. |
| **SAO** | *Sample Adaptive Offset* — HEVC in-loop filter executed after deblocking. |
| **SDMA / VDMA** | *System DMA* (metadata, `mbinfo`, context rows) and *Video DMA* (pixel stripes, AFBC, rotation, downscale). |
| **SR** | RASC CPU *Status Register* (read/written via `mfsr`/`mtsr`). |
| **TRF** | *Transform* — internal transform (DCT/DST/integer) and quantization/dequantization block. |
| **V4L2 M2M** | *Video4Linux2 Memory-to-Memory* — Linux interface exposed by the `mvx` driver (`/dev/video*`) to video applications. |

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
