<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# LSID Contexts, Hardware Scheduler, and Multi-Core

This document specifies the session context model (**LSID — Local Session ID**), the hardware mapping/unmapping lifecycle, the job queue (`JOBQUEUE`), the interrupt flow between host and VPU, and how multi-core topology (2 cores on K1, 4 cores on K3) is driven by the hardware scheduler.

---

## 1. LSID Model and Session Multiplexing (`[P]`)

The Linlon V5 VPU has **`NLSID = 4`** independent session register banks (`LSID 0..3`). Each LSID encapsulates:
- Its own MMU root (`MMU_CTRL`), completely isolating the session's 32-bit virtual address space;
- Its core constraints (`CTRL`: `DISALLOW` + `MAXCORES`);
- Its independent interrupt channels (`LIRQVE` and `IRQHOST`);
- Its AXI bus attributes (`BUSATTR_0..3`) and protection state (`ALLOC`/`NPROT`).

When more than 4 V4L2 sessions are open simultaneously, the driver scheduler multiplexes sessions across the 4 hardware LSIDs by saving and restoring each session's **PCB (*Process Control Block*)**:

```c
struct mvx_lsid_pcb {
    uint32_t ctrl;
    uint32_t mmu_ctrl;
    uint32_t irqhost;
    uint32_t nprot;
};
```

---

## 2. LSID Lifecycle Hardware Sequences (`[P]`)

### 2.1 Mapping a Session to an LSID (`mvx_lsid_map`)

Precondition: the chosen LSID must be free (`ALLOC == 0`).

1. Read `ALLOC` (`+0x0C`) and confirm it equals `MVE_ALLOC_FREE` (`0`).
2. Write `ALLOC`:
   - `MVE_ALLOC_PROTECTED` (`2`) if `pcb->nprot == 0` (protected session), or
   - `MVE_ALLOC_NON_PROTECTED` (`1`) otherwise.
3. Ensure a clean state by executing `TERMINATE`:
   - Write `1` to `TERMINATE` (`+0x18`).
   - Poll `TERMINATE` until it reads `0`.
4. Configure core topology in `CTRL` (`+0x00`) = `pcb->ctrl` (`(maxcores << 8) | disallow`).
5. Configure the L1 page table in `MMU_CTRL` (`+0x04`) = `pcb->mmu_ctrl`.
6. Invalidate the TLB by writing `0` to `FLUSH_ALL` (`+0x10`).
7. Configure `BUSATTR_0..3` (`+0x30..+0x3C`): values from Device Tree (if present) or driver defaults (`0x00, 0x00, 0x33, 0x33`).
8. Restore interrupt state:
   - `LIRQVE` (`+0x1C`) = `0`.
   - `IRQHOST` (`+0x20`) = `pcb->irqhost`.
9. Execute a write memory barrier (`wmb()`) to guarantee visibility of all registers before enabling the scheduler.
10. Enable scheduling by writing `1` to `SCHED` (`+0x14`).

### 2.2 Checking LSID Idle State (`mvx_lsid_idle`)

An LSID is considered **idle** (and therefore safe to unmap) only when **both** conditions hold:
1. The LSID number does **not** appear in any of the 4 slots of `JOBQUEUE` (`0x14`).
2. The LSID number does **not** appear in any of the `NCORES` nibbles of `CORELSID` (`0x10`).

### 2.3 Unmapping an LSID (`mvx_lsid_unmap`)

Mandatory precondition: the LSID must be idle (`mvx_lsid_idle == true`).

1. Disable scheduling by writing `0` to `SCHED` (`+0x14`).
2. Save into the session's PCB: `ctrl`, `mmu_ctrl`, `irqhost`, and `nprot`.
3. Release the LSID by writing `MVE_ALLOC_FREE` (`0`) to `ALLOC` (`+0x0C`).

### 2.4 Forced Termination (`mvx_lsid_terminate`)

Used on session close or hang/watchdog recovery:
1. Remove the LSID from `JOBQUEUE` (if present).
2. Write `1` to `TERMINATE` (`+0x18`).
3. Poll `TERMINATE` until hardware clears it back to `0`.

---

## 3. `JOBQUEUE` Operation (`[P]`)

### 3.1 Enqueuing (`mvx_lsid_jobqueue_add`)

1. Write `0` to `ENABLE` (`0x04`) to pause the global scheduler.
2. Read `JOBQUEUE` (`0x14`).
3. Check whether the LSID is already in any of the 4 slots (`0..3`); if so, skip directly to step 5.
4. Find the first slot whose `LSID` field (`bits [3:0]`) is `0xF` (`MVE_JOBQUEUE_JOB_INVALID`):
   - Fill the slot with `(lsid & 0xF) | (((ncores - 1) & 0xF) << 4)`.
   - Write the updated value to `JOBQUEUE`.
5. Write `1` to `ENABLE` (`0x04`) to resume the global scheduler.

### 3.2 Removal (`mvx_lsid_jobqueue_remove`)

1. Write `0` to `ENABLE`.
2. Read `JOBQUEUE`, compact the slots whose LSID does not match the removed one while preserving FIFO order, and fill the remaining slots with `LSID = 0xF` and `NCORES-1 = ncores - 1`.
3. Write `JOBQUEUE` and re-enable `ENABLE = 1`.

---

## 4. Host ↔ VPU Interrupt Flow (`[P][O]`)

### 4.1 Host → Firmware

When the driver writes new messages/buffers into the `*_HOST` queues or responds to an RPC:
1. Executes a write barrier (`wmb()`) and DMA cache sync (`dma_sync_single_for_device`).
2. Updates `in_wpos` (or `rpc_area->state = MVE_RPC_STATE_RETURN`) with another barrier/sync.
3. Writes **`1` to `IRQHOST`** (`LSID_BASE + 0x20`), waking the cores assigned to the LSID.

### 4.2 Firmware → Host

When the firmware writes responses to `MSG_OUTQ`, returns buffers to `BUF_INRQ`/`BUF_OUTRQ`, or requests an RPC (`state = MVE_RPC_STATE_PARAM`):
1. The firmware ensures memory visibility (by writing `0` to `CR14`, see [internal-mmio-map.md](internal-mmio-map.md)).
2. The firmware writes **`1` to `0xFF02B028`** on the core's internal MMIO bus (`[O]`).
3. Hardware sets the bit corresponding to the LSID in `IRQVE` (`0x18`) and asserts IRQ `74` to the RISC-V CPU.
4. In the *top half* (`irq_top`):
   - The host reads `NLSID` and `IRQVE`.
   - For each LSID with an active bit in `IRQVE`, it writes **`0` to `LIRQVE`** (`LSID_BASE + 0x1C`), executes `wmb()`, marks the LSID in a pending bitmap, and schedules the workqueue (`irq_bottom`).
5. In the *bottom half* (`irq_bottom`):
   - For each marked LSID, the host reads and processes all pending messages in the output queues, handles RPC requests, and drains `ram_print`.

---

## 5. Context Switching (`SWITCH` / `SWITCHED_IN` / `SWITCHED_OUT`) (`[P][O]`)

Because hardware supports multiple LSIDs competing for the same physical cores, the firmware must cooperate with the host scheduler:

```text
  [Free Core] ──(JOBQUEUE + SCHED=1)──► Firmware boots / resumes LSID
                                              │
                                              ▼
                                 Sends MVE_RESPONSE_CODE_SWITCHED_IN {core}
                                              │
                                              ▼
                                 Processes messages, buffers, and frames
                                              │
                        ┌─────────────────────┴─────────────────────┐
                        │ (No remaining work)                        │ (Host sends SWITCH)
                        ▼                                            ▼
             Sends MVE_RESPONSE_CODE_IDLE                 Drains/saves LSID context
                        │                                            │
                        ▼                                            │
             Host replies MVE_REQUEST_CODE_IDLE_ACK                  │
             and/or MVE_REQUEST_CODE_SWITCH                          │
                        │                                            │
                        └─────────────────────┬─────────────────────┘
                                              ▼
                       Sends MVE_RESPONSE_CODE_SWITCHED_OUT {core, reason, sub_reason}
                                              │
                                              ▼
                       Core releases LSID (CORELSID returns to 0xF) and enters wait
```

- **Observed on hardware (`[O]`):** Upon finishing processing of available buffers in an H.264 session, the firmware sends `IDLE`, the host sends `SWITCH`, and the firmware replies with `SWITCHED_OUT(core=0, reason=1, sub_reason=0)`, releasing the core (`CORELSID` returns to `0x0000_00FF` on K1). When new work arrives (e.g., `STOP`/`FLUSH` or new buffers), the host re-inserts the LSID into `JOBQUEUE` and the firmware emits `SWITCHED_IN` again.

---

## 6. Multi-Core Topology Transparency (K1 2 Cores vs. K3 4 Cores) (`[P][O]`)

The firmware **has no access** to the global register block (`0xC050_0000`) and does not read `NCORES`. All adaptation between K1 (2 cores) and K3 (4 cores) happens through four host-driven mechanisms:

1. **MMU instance mapping:** The driver maps the same executable code (`text`) and the same shared region (`master_rw`) into each core instance `0 .. ncores-1` (`0x0000_0000`, `0x0100_0000`, `0x0200_0000`, `0x0300_0000`), allocating distinct **private** BSS pages for each instance.
2. **LSID `CTRL`:** Sets `MAXCORES` and `DISALLOW` according to the SoC's `NCORES`.
3. **`JOBQUEUE`:** Passes `(ncores - 1)` in the hardware scheduling slot.
4. **`MVE_REQUEST_CODE_JOB` message:** Passes the `uint16_t cores` field (between `1` and `NCORES`) to the firmware. The firmware validates `cores >= 1` and coordinates the participating cores of the job using the shared `master_rw` region. Dynamically changing the core count during an active job is not supported.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
