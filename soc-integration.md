<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# SoC Integration — SpacemiT K1 and SpacemiT K3

This document describes how the Linlon V5 VPU integrates into the **SpacemiT K1** and **SpacemiT K3** SoCs: Device Tree nodes, platform resources, clocks, hardware reset, power management (PM/DVFS), and the **mandatory physical DMA address translation** on K1.

---

## 1. Platform Resources and Device Tree (`[P][O]`)

### 1.1 SpacemiT K1 (`k1-x.dtsi`)

```dts
linlon-v5@c0500000 {
    compatible = "arm china,linlon-v5";
    reg = <0x0 0xC0500000 0x0 0x10000>;     /* 64 KiB mapped in DT (512 KiB in SoC map) */
    interrupt-parent = <&intc>;
    interrupts = <74>;
    clocks = <&ccu CLK_VPU>;
    clock-names = "vpu_clk";
    power-domains = <&power K1X_PMU_VPU_PWR_DOMAIN>;
    resets = <&reset RESET_VPU>;
    interconnects = <&dram_range1>;
    interconnect-names = "dma-mem";
    clk,pm-runtime,no-sleep;
    status = "okay";
};
```

### 1.2 Resource Comparison Table (K1 vs. K3)

| Resource | SpacemiT K1 | SpacemiT K3 | Provenance |
|---|---|---|---|
| `compatible` | `"arm china,linlon-v5"` | `"arm china,linlon-v5"` | `[P][O]` |
| MMIO Base (APB) | `0x0000_0000_C050_0000` | `0x0000_0000_C050_0000` | `[P][O]` |
| Reserved window in SoC map | `512 KiB` (`0x80000`) | `512 KiB` (`0x80000`) | `[P]` |
| `reg` window in DT | `64 KiB` (`0x10000`) | `8 KiB` (`0x2000`) | `[P][O]` |
| Interrupt (IRQ) | `74` (`0x4A`, `sys_int_ap[74]`) | `74` (`0x4A`, level-high) | `[P][O]` |
| IRQ registration flag | `IRQF_SHARED` | `IRQF_SHARED` | `[P]` |
| Clock | `CLK_VPU` (`vpu_clk`) | `vpu_clk` (parent `pll2_d3`) | `[P][O]` |
| Reset | `RESET_VPU` | VPU domain reset | `[P]` |
| Power domain | `K1X_PMU_VPU_PWR_DOMAIN` | VPU PMU domain | `[P]` |
| Device DMA mask | `DMA_BIT_MASK(34)` (34-bit) | ≥ 34-bit | `[P]` |
| Optional `busattr` property | Absent in DT (uses driver default) | `0x33` observed in `BUSATTR_0..3` | `[P][O]` |

> **Note on legacy compatibles (`[P]`):** The `mvx` driver also declares legacy compatibles (`arm,mali-mve`, `arm,mali-v500`, `arm,mali-v550`, `arm,mali-v61`, `arm china,linlon-v7`), but both K1 and K3 use `"arm china,linlon-v5"`.

---

## 2. Initialization (Probe) and Hardware Reset Sequence (`[P]`)

### 2.1 Host Probe Sequence

1. Map the MMIO region (`0xC050_0000`) and obtain IRQ `74`.
2. Configure coherent and streaming DMA mask to **34 bits** (`DMA_BIT_MASK(34)`).
3. Enable `pm_runtime` and acquire the `vpu_clk` clock and `RESET_VPU` reset control.
4. Deassert the block reset (`reset_control_deassert`).
5. Set the nominal clock frequency:
   - **K1:** `819,200,000 Hz` (`819.2 MHz`).
   - **K3:** `1,000,000,000 Hz` (`1 GHz`).
6. Read `HARDWARE_ID` (`0x00`), validate family (`0x5662`/`0x5663` → `MVE_v52_v76`), read `FUSE` (`0x34`), `NCORES` (`0x08`), and `NLSID` (`0x0C`).
7. Initialize scheduler/LSID structures and register the interrupt handler (`IRQF_SHARED`) with a workqueue for the *bottom half*.

### 2.2 Internal Block Reset Sequence (`reset_hw`)

Executed on power-up (*runtime resume* / *system resume*) before reactivating the scheduler:

1. Read `ncores = readl(BASE + 0x08)` (`NCORES`).
2. Build the expected idle-cores mask (`corelsid_mask`): one `0xF` nibble per active core:
   - K1 (`ncores = 2`): `corelsid_mask = 0x0000_00FF`.
   - K3 (`ncores = 4`): `corelsid_mask = 0x0000_FFFF`.
3. Write `1` to `RESET` (`BASE + 0x50`).
4. Poll `CORELSID` (`BASE + 0x10`) until `readl(CORELSID) == corelsid_mask` (ensuring all cores have exited execution and returned to the free `0xF` state).
5. Write `0` to `CLKFORCE` (`BASE + 0x24`) to release automatic block *clock gating*.

> **Critical precondition (`[P]`):** The reset is only considered complete when **all** `CORELSID` nibbles corresponding to the physical `NCORES` read `0xF`.

---

## 3. Clocks, Power, and DVFS (`[P][O]`)

### 3.1 Frequency Table (OPPs)

| Level (`vmin`) | Frequency (Hz) | Available on K1 | Available on K3 |
|---:|---:|:---:|:---:|
| 0 | `307,200,000` (307.2 MHz) | Yes | Yes |
| 1 | `409,600,000` (409.6 MHz) | Yes | Yes |
| 2 | `491,520,000` (491.52 MHz) | Yes | Yes |
| 3 | `600,000,000` (600.0 MHz) | Yes | Yes |
| 4 | `614,400,000` (614.4 MHz) | Yes | Yes |
| 5 | `750,000,000` (750.0 MHz) | Yes | Yes |
| 6 | **`819,200,000` (819.2 MHz)** | **Yes (K1 max)** | Yes |
| 7 | **`1,000,000,000` (1.0 GHz)** | No | **Yes (K3 max)** |

### 3.2 Power Management (Runtime PM / System PM)

- **Power-on / Runtime Resume:** Enables `vpu_clk` → executes `reset_hw()` → resumes the scheduler (`mvx_sched_resume`).
- **Power-off / Runtime Suspend:** Suspends the scheduler (`mvx_sched_suspend`, waiting for sessions to drain) → disables `vpu_clk`.
- The `clk,pm-runtime,no-sleep` property in the K1 DT signals that the clock should not be aggressively gated off while scheduler/session state is active.
- **Firmware role:** The firmware **does not control** clock, voltage, reset, or power domain. It only supports internal power saving via the `wait` instruction (RASC CPU halt until interrupt) and internal *clock gating* control (optionally disableable via `MVE_OPT_DISABLE_FEATURE_SLEEP`).

---

## 4. Physical DMA Address Translation on SpacemiT K1 (`[P]`)

In the SpacemiT K1 SoC physical memory map, the RISC-V CPU sees an **IO** window between `0x00_8000_0000` and `0x00_FFFF_FFFF` (2 GiB), splitting DRAM into two regions in the CPU's physical address space:
- Low DRAM: `0x00_0000_0000` – `0x00_7FFF_FFFF` (first 2 GiB)
- IO window (CPU): `0x00_8000_0000` – `0x00_FFFF_FFFF`
- High DRAM (CPU): `0x01_0000_0000` – `0x04_7FFF_FFFF` (above 2 GiB)

The VPU, however, sees DRAM **contiguously** on its AXI bus (without the 2 GiB IO hole), from `0x00_0000_0000` through `0x03_FFFF_FFFF`:

```text
CPU Physical Address Space (RISC-V)        VPU Physical Address Space (AXI / PTE)
───────────────────────────────────        ──────────────────────────────────────
0x04_8000_0000 ┌───────────┐               0x04_0000_0000 ┌───────────┐
               │           │                              │           │
               │ High DRAM │  ◄── (- 0x8000_0000) ──►     │ High DRAM │
               │           │                              │(contiguous)│
0x01_0000_0000 ├───────────┤               0x00_8000_0000 ├───────────┤
               │    IO     │  (invisible to VPU)          │           │
0x00_8000_0000 ├───────────┤                              │ Low DRAM  │
               │ Low DRAM  │  ◄── (identity) ───────►     │           │
0x00_0000_0000 └───────────┘               0x00_0000_0000 └───────────┘
```

### Normative Conversion Formulas (Host Side)

- **CPU → VPU (for populating MMU PTEs and `MMU_CTRL`):**
  - If `pa_cpu >= 0x1_0000_0000`: `pa_vpu = pa_cpu - 0x8000_0000`
  - Otherwise: `pa_vpu = pa_cpu`
- **VPU → CPU (for converting a PA extracted from a PTE back to a kernel address):**
  - If `pa_vpu >= 0x8000_0000`: `pa_cpu = pa_vpu + 0x8000_0000`
  - Otherwise: `pa_cpu = pa_vpu`

> **Impact on firmware:** This translation is 100% transparent to the RASC firmware, because the firmware operates exclusively on 32-bit virtual addresses (`mvx_mmu_va`) translated by the MMU. However, any host driver or bench tool on K1 that builds page tables for the VPU **must** apply `phys_cpu2vpu` to all physical addresses (L1 tables, L2 tables, and data/code pages), or the VPU will access invalid memory or trigger `MMU ABORT`.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
