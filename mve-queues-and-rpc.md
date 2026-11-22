<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# MVE 3.5 Protocol — Circular Queues, RPC, and Log RAM

This document specifies the shared-memory transport layer of the **MVE 3.5** protocol (`mve_protocol_def.h` and `mvx_firmware_v2.c`, `[P]`): the 6 circular 4 KiB rings, packing and wrap-around rules, the synchronous RPC channel, and the circular RAM log buffer (`ram_print`).

---

## 1. Channel Virtual Address Map (`[P]`)

All channels below are mapped by the host driver with `MVX_ATTR_SHARED_RW` and `MVX_ACCESS_READ_WRITE` at identical fixed virtual addresses across all cores of the session:

| Channel | Fixed VA | Size | Structure | Primary Writer |
|---|---|---:|---|---|
| `MVE_COMM_MSG_INQ_ADDR` | `0x1007_9000` | 4 KiB | `struct mve_comm_area_host` | Host (requests → FW) |
| `MVE_COMM_MSG_OUTQ_ADDR` | `0x1007_A000` | 4 KiB | `struct mve_comm_area_mve` | Firmware (responses → Host) |
| `MVE_COMM_BUF_INQ_ADDR` | `0x1007_B000` | 4 KiB | `struct mve_comm_area_host` | Host (input buffers → FW) |
| `MVE_COMM_BUF_INRQ_ADDR` | `0x1007_C000` | 4 KiB | `struct mve_comm_area_mve` | Firmware (returned input buffers → Host) |
| `MVE_COMM_BUF_OUTQ_ADDR` | `0x1007_D000` | 4 KiB | `struct mve_comm_area_host` | Host (output buffers → FW) |
| `MVE_COMM_BUF_OUTRQ_ADDR` | `0x1007_E000` | 4 KiB | `struct mve_comm_area_mve` | Firmware (returned output buffers → Host) |
| `MVE_COMM_RPC_ADDR` | `0x1007_F000` | 4 KiB | `struct mve_rpc_communication_area` | Bidirectional (handshake via `state`) |
| `MVE_FW_PRINT_RAM_ADDR` | `0x1010_0000` | 512 KiB | `struct mve_fw_ram_print_head_aera` + slots | Firmware writes `wr_cnt`/slots; Host updates `rd_cnt` |

---

## 2. Circular Queue Layout (`[P]`)

Each channel pair (Messages, Input Buffers, Output Buffers) consists of **two 4 KiB pages** — one page written exclusively by the Host and the other written exclusively by the Firmware — eliminating false sharing and concurrent write conflicts:

```c
#define MVE_COMM_QUEUE_SIZE_IN_WORDS 1020

/* Page written ONLY by the Host (0x10079000, 0x1007B000, 0x1007D000) */
struct mve_comm_area_host {
    volatile uint16_t out_rpos;      /* offset 0x00: word index that Host has already read from mve->out_data */
    volatile uint16_t in_wpos;       /* offset 0x02: word index up to which Host has written in host->in_data */
    volatile uint32_t reserved[3];   /* offset 0x04..0x0F: reserved (12 bytes) */
    volatile uint32_t in_data[1020]; /* offset 0x10..0xFFF: circular ring of 1020 words (4080 bytes) */
};

/* Page written ONLY by the Firmware (0x1007A000, 0x1007C000, 0x1007E000) */
struct mve_comm_area_mve {
    volatile uint16_t out_wpos;      /* offset 0x00: word index up to which FW has written in mve->out_data */
    volatile uint16_t in_rpos;       /* offset 0x02: word index that FW has already read from host->in_data */
    volatile uint32_t reserved[3];   /* offset 0x04..0x0F: reserved (12 bytes) */
    volatile uint32_t out_data[1020];/* offset 0x10..0xFFF: circular ring of 1020 words (4080 bytes) */
};
```

### 2.1 Message Header (`struct mve_msg_header`, 4 Bytes)

Every entry in any of the 6 queues begins with a 32-bit word containing:

```c
struct mve_msg_header {
    uint16_t code; /* bits [15:0]: message code (1001..1013, 2001..2019, 3001..3004) */
    uint16_t size; /* bits [31:16]: size in BYTES of the payload following the header (0 if empty) */
};
```

> **Versioning rule (`[P]`):** The consumer **must always** consume `size` bytes (rounded up to 32-bit words) based on `header.size`, never assuming a fixed `sizeof(struct)`, in order to tolerate future payload extensions.

### 2.2 Normative Circular Ring Rules (`[P]`)

1. **Indexing unit:** All pointers (`in_wpos`, `in_rpos`, `out_wpos`, `out_rpos`) are indices in **32-bit words** in the range `[0, 1019]` (`modulo 1020`).
2. **32-bit alignment:** A message of `size` bytes consumes exactly:
   ```text
   words_consumed = 1 + ceil(size / 4) = (4 + size + 3) / 4
   ```
   If `size` is not a multiple of 4, the trailing `1..3` bytes occupy the start of the last word and the index advances by 1 full word (rounded up).
3. **Word-by-word wrap-around:** Each word (of both the header and the payload) is read/written at `data[pos]` with `pos = (pos + 1) % 1020`. A message may cleanly wrap across the `1019 → 0` boundary.
4. **Computing available space/data:**
   - **Words available to read (consumer):**
     ```c
     int avail = (int)wpos - (int)rpos;
     if (avail < 0) avail += 1020;
     /* avail == 0 means queue is empty */
     ```
   - **Free space to write (producer):**
     ```c
     int free_words = (int)rpos - (int)wpos;
     if (free_words <= 0) free_words += 1020;
     /* Requires free_words > words_consumed (always keep >= 1 word free to distinguish full from empty!) */
     ```
5. **Update ordering and barriers (Firmware side):**
   - **When sending (FW → Host):**
     1. Write `header` and payload into `mve->out_data[]`.
     2. Update `mve->out_wpos = new_wpos`.
     3. Execute synchronization `CR14 = 0` (`mtcr r, 0x0E`).
     4. Trigger an IRQ to the host by writing `1` to `0xFF02B028` (`[O]`).
   - **When consuming (Host → FW):**
     1. Read `host->in_wpos`; if different from `mve->in_rpos`, read `header` and `header.size` bytes from `host->in_data[]`.
     2. Update `mve->in_rpos = new_rpos` and execute `CR14 = 0` so the host observes the freed space.
6. **Code validation by the host driver (`[P]`):** The driver discards reads whose `header.code` lies outside `[1001..1013]`, `[2001..2019]`, or `[3001..3004]`.

---

## 3. RPC Channel (`0x1007_F000`) (`[P][O]`)

The RPC channel allows the firmware to request synchronous services from the host driver — primarily allocating, resizing, and freeing memory in the session MMU (since the firmware has no physical allocator of its own) and debug printing.

### 3.1 `struct mve_rpc_communication_area` (1024 Bytes)

```c
#define MVE_RPC_AREA_SIZE_IN_WORDS 256
#define MVE_RPC_DATA_SIZE_IN_WORDS 253

struct mve_rpc_communication_area {
    volatile uint32_t state;   /* offset 0x00: 0=FREE, 1=PARAM, 2=RETURN */
    volatile uint32_t call_id; /* offset 0x04: 1=DEBUG_PRINTF, 2=MEM_ALLOC, 3=MEM_RESIZE, 4=MEM_FREE */
    volatile uint32_t size;    /* offset 0x08: size in bytes of data in params */
    union mve_rpc_params {     /* offset 0x0C..0x3FF (1012 bytes = 253 words) */
        volatile uint32_t data[253];
        struct {
            char string[1012]; /* NUL-terminated ASCII string */
        } debug_print;
        struct {
            uint32_t size;           /* +0x00: desired initial size in bytes */
            uint32_t max_size;       /* +0x04: maximum capacity for future MEM_RESIZE calls */
            uint8_t  region;         /* +0x08: 0=MVE_MEM_REGION_PROTECTED, 1=MVE_MEM_REGION_OUTBUF/FRAMEBUF */
            uint8_t  log2_alignment; /* +0x09: minimum alignment = 2^(log2_alignment) bytes */
        } mem_alloc;                 /* expected size = 12 bytes */
        struct {
            uint32_t ve_pointer;     /* +0x00: VA previously returned by MEM_ALLOC */
            uint32_t new_size;       /* +0x04: new size in bytes */
        } mem_resize;                /* expected size = 8 bytes */
        struct {
            uint32_t ve_pointer;     /* +0x00: VA to free */
        } mem_free;                  /* expected size = 4 bytes */
    } params;
};
```

### 3.2 RPC Handshake Protocol (State Machine)

```text
  [state = FREE (0)]
         │
         │ Firmware populates call_id, size, params
         │ Firmware writes state = PARAM (1) + CR14 = 0 + IRQ (0xFF02B028 = 1)
         ▼
  [state = PARAM (1)]
         │
         │ Host services the call in irq_bottom:
         │   - DEBUG_PRINTF (1): prints params.debug_print.string to kernel log
         │   - MEM_ALLOC    (2): allocates pages, maps in MMU (region + alignment),
         │                       writes size = 4, params.data[0] = VA (or 0 on failure)
         │   - MEM_RESIZE   (3): resizes pages, performs MMU FLUSH_ALL,
         │                       writes size = 4, params.data[0] = VA (or 0 on failure)
         │   - MEM_FREE     (4): unmaps and frees pages, performs MMU FLUSH_ALL,
         │                       writes size = 0
         │ Host executes wmb() + state = RETURN (2) + dma_sync + IRQHOST = 1
         ▼
  [state = RETURN (2)]
         │
         │ Firmware reads result (params.data[0] for ALLOC/RESIZE)
         │ Firmware writes state = FREE (0) before the next RPC call
         ▼
  [state = FREE (0)]
```

### 3.3 Real Hardware Example Observed (`[O]`)

During an H.264 128×128 session on K1:
- **Firmware Request:** `state = 1 (PARAM)`, `call_id = 2 (MEM_ALLOC)`, `size = 12`, `mem_alloc = { size = 0x20000 (128 KiB), max_size = 0x4000000 (64 MiB), region = 0 (PROTECTED), log2_alignment = 22 (4 MiB) }`.
- **Host Response:** `state = 2 (RETURN)`, `call_id = 2`, `size = 4`, `params.data[0] = 0x20400000` (4 MiB-aligned VA inside `PROTECTED`).

---

## 4. Firmware Circular Log RAM (`0x1010_0000`, 512 KiB) (`[P]`)

The driver maps 512 KiB contiguously at `MVE_FW_PRINT_RAM_ADDR` (`0x1010_0000`) for asynchronous firmware diagnostic messages (read by the driver when `MVX_FW_DEBUG_ENABLE` is active):

### 4.1 Header (`struct mve_fw_ram_print_head_aera`, 128 Bytes at `0x1010_0000`)

| Offset | Type | Field | Writer | Function |
|---:|---|---|---|---|
| `0x00` | `volatile uint32_t` | `rd_cnt` | Host | Monotonic counter of messages already read by the host |
| `0x04` | `uint32_t[15]` | `reserved0` | — | Padding to separate cache line (64 B) |
| `0x40` | `volatile uint32_t` | `flag` | Firmware | Validation magic number: **`0x11223356`** (`RAM_PRINT_FLAG`) |
| `0x44` | `volatile uint32_t` | `index` | Firmware | Current write index (`0 .. RAM_PRINT_BUF_CNT - 1`) |
| `0x48` | `volatile uint32_t` | `wr_cnt` | Firmware | Monotonic counter of messages written by the firmware |
| `0x4C` | `uint32_t[13]` | `reserved1` | — | Padding up to 128 bytes (`0x80`) |

### 4.2 Message Slots (`0x1010_0080` – `0x1017_FFFF`)

- **Size of each slot:** `RAM_PRINT_MAX_LEN = 128` bytes (NUL-terminated ASCII string).
- **Total number of slots:** `RAM_PRINT_BUF_CNT = (524288 / 128) - 1 = 4095` slots (`slot 0` at `0x1010_0080`, `slot k` at `0x1010_0080 + k * 128`).
- **Firmware write rule:**
  1. Initialize `flag = 0x11223356`, `index = 0`, `wr_cnt = 0` at boot.
  2. To log a line: copy text (max 127 chars + `\0`) into slot `idx = wr_cnt % 4095` (address `0x1010_0080 + idx * 128`), update `index = idx`, and increment `wr_cnt` (with a barrier before incrementing `wr_cnt`).
  3. The driver reads all entries from `rd_cnt` to `wr_cnt` whenever `flag == 0x11223356` and `index < 4095`, updating `rd_cnt` after draining.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
