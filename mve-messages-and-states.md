<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# MVE 3.5 Protocol — Control Messages, Events, and State Machine

This document catalogs all control messages (Host → Firmware requests and Firmware → Host responses), asynchronous events, error codes, and the formal state machine of an MVE 3.5 session (`mve_protocol_def.h`, `[P][O]`).

---

## 1. Request Catalog (Host → Firmware in `MSG_INQ`) (`[P]`)

| Code | Public Macro | Payload (`header.size`) | Semantics and Preconditions |
|---:|---|---|---|
| `1001` | `MVE_REQUEST_CODE_GO` | empty (`0`) | Requests transition to `RUNNING` (`2`). Reject if a transition is already pending or if no active job is in the job queue. Responds with `STATE_CHANGE(RUNNING)`. |
| `1002` | `MVE_REQUEST_CODE_STOP` | empty (`0`) | Requests transition to `STOPPED` (`0`). Drains in-flight pipeline work and responds with `STATE_CHANGE(STOPPED)`. Reject if a transition is pending. |
| `1003` | `MVE_REQUEST_CODE_INPUT_FLUSH` | empty (`0`) | **Only allowed in `STOPPED`.** Returns all pending buffers from the input queue via `BUF_INRQ` (sending `RESPONSE_CODE_INPUT` for each) and concludes with `INPUT_FLUSHED`. |
| `1004` | `MVE_REQUEST_CODE_OUTPUT_FLUSH` | empty (`0`) | **Allowed in `STOPPED` or in the wait state after `SEQUENCE_PARAMETERS`.** Returns all held output buffers via `BUF_OUTRQ` (with `RESPONSE_CODE_OUTPUT`) and concludes with `OUTPUT_FLUSHED`. |
| `1005` | `MVE_REQUEST_CODE_SWITCH` | empty (`0`) | Requests that the firmware save LSID state and release physical cores; upon completion, responds with `SWITCHED_OUT` and enters `wait`. |
| `1006` | `MVE_REQUEST_CODE_PING` | empty (`0`) | Health check. Must respond with `MVE_RESPONSE_CODE_PONG` (`2009`). |
| `1008` | `MVE_REQUEST_CODE_DUMP` | empty (`0`) | Requests an internal state dump for diagnostics (e.g., `ram_print`). **Forbidden in protected sessions** (must return error). Upon completion, responds with `MVE_RESPONSE_CODE_DUMP` (`2012`). |
| `1009` | `MVE_REQUEST_CODE_JOB` | `struct mve_request_job` (8 B) | Submits a job specifying `cores`, `frames`, and `flags`. Once accepted/dequeued on all cores of the session, responds with `JOB_DEQUEUED`. |
| `1010` | `MVE_REQUEST_CODE_SET_OPTION` | `struct mve_request_set_option` (variable) | **Only allowed in `STOPPED`.** Configures a session option (see [mve-options.md](mve-options.md)). Responds with `SET_OPTION_CONFIRM` or `SET_OPTION_FAIL` (non-fatal). |
| `1011` | `MVE_REQUEST_CODE_RELEASE_REF_FRAME` | `struct mve_request_release_ref_frame` (4 B) | **Decode only.** Asks the firmware to release a buffer previously returned with `REF_FRAME` as soon as it is no longer used as a reference. When released, responds with `REF_FRAME_UNUSED`. |
| `1012` | `MVE_REQUEST_CODE_IDLE_ACK` | empty (`0`) | Host acknowledgment (MVE >= 3.1) in response to an `MVE_RESPONSE_CODE_IDLE` previously sent by the firmware. Error if received without a pending `IDLE`. |
| `1013` | `MVE_REQUEST_CODE_DEBUG` | `uint32_t level` (4 B) | Sets firmware log level: `0` = disabled, `1` = `FW_LOG_ERROR`, `2` = `WARNING`, `3` = `INFO`, `4` = `DEBUG`, `5` = `VERBOSE`. Responds with `MVE_RESPONSE_CODE_DEBUG` (`2019`). |

### 1.1 Request Structures

```c
struct mve_request_job {
    uint16_t cores;  /* >= 1; number of cores for the session (must match HW scheduler) */
    uint16_t frames; /* number of frames to process; 0 = infinite */
    uint32_t flags;  /* bit 0: MVE_JOB_FLAG_DISABLE_BNDMGR (0x01) */
};

struct mve_request_release_ref_frame {
    uint32_t buffer_address; /* VA of the AFBC reference buffer */
};
```

**Mandatory validations on `JOB` (`[P][D]`):** `cores` must not be `0` nor exceed the configured/supported limit (`1..8`); dynamically changing `cores` between active jobs is unsupported; unknown flags must be rejected; if there is no room in the internal job queue, report an error.

---

## 2. Response Catalog (Firmware → Host in `MSG_OUTQ`) (`[P]`)

| Code | Public Macro | Payload (`header.size`) | Semantics |
|---:|---|---|---|
| `2001` | `MVE_RESPONSE_CODE_SWITCHED_IN` | `struct mve_response_switched_in` (4 B) | Sent when the firmware boots or resumes execution on a core |
| `2002` | `MVE_RESPONSE_CODE_SWITCHED_OUT` | `struct mve_response_switched_out` (12 B) | Sent when the last core of the session completes switch-out |
| `2003` | `MVE_RESPONSE_CODE_SET_OPTION_CONFIRM` | `struct mve_response_set_option_confirm` (4 B) | Confirms successful application of option `index` |
| `2004` | `MVE_RESPONSE_CODE_JOB_DEQUEUED` | `struct mve_response_job_dequeued` (4 B) | All cores of the session have dequeued the job (`valid_job = 1` if valid) |
| `2005` | `MVE_RESPONSE_CODE_INPUT` | empty (`0`) | Notifies that an input buffer has been returned in `BUF_INRQ` (`0x1007_C000`) |
| `2006` | `MVE_RESPONSE_CODE_OUTPUT` | empty (`0`) | Notifies that an output buffer has been returned in `BUF_OUTRQ` (`0x1007_E000`) |
| `2007` | `MVE_RESPONSE_CODE_INPUT_FLUSHED` | empty (`0`) | Completes `INPUT_FLUSH` |
| `2008` | `MVE_RESPONSE_CODE_OUTPUT_FLUSHED` | empty (`0`) | Completes `OUTPUT_FLUSH` |
| `2009` | `MVE_RESPONSE_CODE_PONG` | empty (`0`) | Response to `PING` |
| `2010` | `MVE_RESPONSE_CODE_ERROR` | `struct mve_response_error` (132 B) | **Fatal session error** (no further operation is guaranteed after emission) |
| `2011` | `MVE_RESPONSE_CODE_STATE_CHANGE` | `struct mve_response_state_change` (4 B) | Confirms state change (`0 = STOPPED`, `2 = RUNNING`) |
| `2012` | `MVE_RESPONSE_CODE_DUMP` | empty (`0`) | Completes `DUMP` |
| `2013` | `MVE_RESPONSE_CODE_IDLE` | empty (`0`) | Informs the host that there is no runnable work (waiting for buffers or idle) |
| `2014` | `MVE_RESPONSE_CODE_FRAME_ALLOC_PARAM` | `struct mve_response_frame_alloc_parameters` (32 B) | **Decode only.** Reports output buffer size requirements |
| `2015` | `MVE_RESPONSE_CODE_SEQUENCE_PARAMETERS` | `struct mve_response_sequence_parameters` (8 B) | **Decode only.** Reports sequence parameters; pauses until `OUTPUT_FLUSH` |
| `2016` | `MVE_RESPONSE_CODE_EVENT` | `struct mve_response_event` (variable) | Informational/diagnostic event (requires no mandatory host response) |
| `2017` | `MVE_RESPONSE_CODE_SET_OPTION_FAIL` | `struct mve_response_set_option_fail` (132 B) | **Non-fatal error** while configuring option `index` (session remains alive) |
| `2018` | `MVE_RESPONSE_CODE_REF_FRAME_UNUSED` | `struct mve_response_ref_frame_unused` (4 B) | Confirms that `unused_buffer_address` is no longer a reference |
| `2019` | `MVE_RESPONSE_CODE_DEBUG` | empty (`0`) | Confirms application of `MVE_REQUEST_CODE_DEBUG` |

### 2.1 Response Structures

```c
struct mve_response_switched_in  { uint32_t core; };
struct mve_response_switched_out { uint32_t core; uint32_t reason; uint32_t sub_reason; };
struct mve_response_state_change { uint32_t new_state; }; /* 0=MVE_STATE_STOPPED, 2=MVE_STATE_RUNNING */
struct mve_response_job_dequeued { uint32_t valid_job; };
struct mve_response_set_option_confirm { uint32_t index; };
struct mve_response_ref_frame_unused   { uint32_t unused_buffer_address; };

/* Note [P][O]: In fw_v3/mve_protocol_def.h the comment next to the macro
 * MVE_RESPONSE_CODE_SET_OPTION_CONFIRM (2003) states "no extra data",
 * however the header itself defines struct mve_response_set_option_confirm
 * and real hardware traces (K1/K3) always transmit the 4 bytes of index
 * (header.size = 4). */

struct mve_response_error {
    uint32_t error_code;
    char     message[128]; /* NUL-terminated ASCII string */
};

struct mve_response_set_option_fail {
    uint32_t index;
    char     message[128];
};
```

#### Error Codes (`mve_response_error.error_code`)

| Code | Public Macro | Cause |
|---:|---|---|
| `1` | `MVE_ERROR_ABORT` | Fatal RASC CPU exception or `MMU ABORT` |
| `2` | `MVE_ERROR_OUT_OF_MEMORY` | Allocation failure (RPC returned 0 or insufficient `IntBuf`) |
| `3` | `MVE_ERROR_ASSERT` | Internal invariant failure (assert) |
| `4` | `MVE_ERROR_UNSUPPORTED` | Unsupported message code or feature |
| `6` | `MVE_ERROR_INVALID_BUFFER` | Invalid buffer descriptor or outside permitted regions |
| `8` | `MVE_ERROR_INVALID_STATE` | Command invalid for the current state (e.g., `INPUT_FLUSH` while `RUNNING`) |
| `9` | `MVE_ERROR_WATCHDOG` | Internal watchdog timer expiration |

### 2.2 Allocation and Sequence Parameters (Decode)

```c
struct mve_response_frame_alloc_parameters {
    uint16_t planar_alloc_frame_width;             /* +0x00 */
    uint16_t planar_alloc_frame_height;            /* +0x02 */
    uint32_t afbc_alloc_bytes;                     /* +0x04 (per field if interlaced) */
    uint32_t afbc_alloc_bytes_downscaled;          /* +0x08 */
    uint16_t afbc_width_in_superblocks;            /* +0x0C */
    uint16_t afbc_width_in_superblocks_downscaled; /* +0x0E */
    uint16_t cropx;                                /* +0x10: X offset for optimal AXI bursts */
    uint16_t cropy;                                /* +0x12: Y offset for optimal AXI bursts */
    uint32_t mbinfo_alloc_bytes;                   /* +0x14 (debug) */
    uint16_t dsl_frame_width;                      /* +0x18 (arbitrary downscale) */
    uint16_t dsl_frame_height;                     /* +0x1A */
    uint16_t dsl_pos_mode;                         /* +0x1C */
    uint8_t  ctu_size;                             /* +0x1E (16 for H.264, 16/32/64 for HEVC) */
    /* +0x1F: 1 byte padding (sizeof = 32) */
};

struct mve_response_sequence_parameters {
    uint8_t interlace;             /* +0x00: 0=progressive, 1=interlaced */
    uint8_t chroma_format;         /* +0x01: 0=MONO, 1=420, 2=422, 3=440, 4=ARGB */
    uint8_t bitdepth_luma;         /* +0x02: 8, 9, or 10 */
    uint8_t bitdepth_chroma;       /* +0x03: 8, 9, or 10 */
    uint8_t num_buffers_planar;    /* +0x04: minimum planar buffers */
    uint8_t num_buffers_afbc;      /* +0x05: minimum AFBC buffers (includes DPB/refs) */
    uint8_t range_mapping_enabled; /* +0x06: VC-1 AP specific */
    uint8_t reserved0;             /* +0x07 */
};
```

---

## 3. Events (`MVE_RESPONSE_CODE_EVENT` = `2016`) (`[P]`)

`struct mve_response_event` is packed (`__attribute__((packed))`): `uint32_t event_code` followed by the `event_data` union:

| `event_code` | Public Macro | Payload in `event_data` |
|---:|---|---|
| `1` | `MVE_EVENT_ERROR_STREAM_CORRUPT` | `char message[128]` — recoverable bitstream corruption |
| `2` | `MVE_EVENT_ERROR_STREAM_NOT_SUPPORTED` | `char message[128]` — unsupported stream feature |
| `3` | `MVE_EVENT_PROCESSED` | `struct mve_event_processed` (84 B) — frame telemetry (if `PROFILING=1`) |
| `4` | `MVE_EVENT_REF_FRAME` | `struct mve_event_ref_frame` (40 B) — AFBC reference metadata (if `ENC_EXPOSE_REF_FRAMES=1`) |
| `5` | `MVE_EVENT_TRACE_BUFFERS` | `struct mve_event_trace_buffers` — RASC addresses/sizes of up to 40 trace buffers |

### 3.1 `struct mve_event_processed` (84 Bytes)

| Offset | Type | Field | Meaning |
|---:|---|---|---|
| `0x00` | `uint8_t` | `pic_format`, `qp`, `pad0`, `pad1` | Picture format and average QP |
| `0x04` | `uint32_t` | `parse_start_time`, `parse_end_time`, `parse_idle_time` | Timestamps and idle time of the *parse* stage |
| `0x10` | `uint32_t` | `pipe_start_time`, `pipe_end_time`, `pipe_idle_time` | Timestamps and idle time of the *pipe* stage (`pipe_idle_time = 0` in decode) |
| `0x1C` | `uint32_t` | `parser_coreid`, `pipe_coreid` | IDs of the cores that executed *parse* and *pipe* |
| `0x24` | `uint32_t` | `bitstream_bits` | Bitstream bits consumed/produced in the frame |
| `0x28` | `uint32_t` | `intermediate_buffer_size`, `total_memory_allocated` | `IntBuf` usage and total allocated memory (including refs) |
| `0x30` | `uint32_t` | `bus_read_bytes`, `bus_write_bytes`, `afbc_bytes` | AXI and AFBC traffic counters |
| `0x3C` | `uint32_t` | `slice0_end_time`, `stream_start_time`, `stream_open_time` | Latency timestamps |

---

## 4. Session State Machine (`[P][O][D]`)

```text
                       ┌────────────────────────┐
       [Boot/Init] ───►│     STOPPED (0)        │◄───────────────────────┐
                       │ • Accepts SET_OPTION   │                        │
                       │ • Accepts INPUT_FLUSH  │                        │
                       │ • Accepts OUTPUT_FLUSH │                        │
                       │ • Accepts JOB / PING   │                        │
                       └───────────┬────────────┘                        │
                                   │ GO (1001)                           │ STOP (1002)
                                   │ → STATE_CHANGE(2)                   │ → STATE_CHANGE(0)
                                   ▼                                     │
                       ┌────────────────────────┐                        │
                       │     RUNNING (2)        │────────────────────────┘
                       │ • Processes IN/OUT bufs│
                       │ • Rejects SET_OPTION   │
                       │ • Rejects INPUT_FLUSH  │
                       │ • Emits IDLE if empty  │
                       └───────────┬────────────┘
                                   │ (Decode: sequence change)
                                   │ → FRAME_ALLOC_PARAM + SEQUENCE_PARAMETERS
                                   ▼
                       ┌────────────────────────┐
                       │   WAIT_OUTPUT_FLUSH    │
                       │ • Pauses production    │
                       │ • Waits OUTPUT_FLUSH   │
                       │   → returns OUT bufs   │
                       │   → OUTPUT_FLUSHED     │
                       └───────────┬────────────┘
                                   │ (Resumes after OUTPUT_FLUSHED and new OUT bufs)
                                   └──────────────► RUNNING (2)
```

### Mandatory State Machine Invariants

1. **Single transition at a time:** If `GO` or `STOP` arrives while a previous transition is still pending, respond with an error (`MVE_ERROR_INVALID_STATE`).
2. **State-specific restrictions:**
   - `SET_OPTION` outside `STOPPED` → `MVE_RESPONSE_CODE_SET_OPTION_FAIL`.
   - `INPUT_FLUSH` outside `STOPPED` → `MVE_ERROR_INVALID_STATE`.
   - `OUTPUT_FLUSH` is only accepted in `STOPPED` or in `WAIT_OUTPUT_FLUSH` (after `SEQUENCE_PARAMETERS`).
3. **Interaction with `IDLE` and `SWITCH`:**
   - In `RUNNING`, when there are no buffers or runnable work, the firmware sends `IDLE` (`2013`).
   - The host (MVE >= 3.1) responds with `IDLE_ACK` (`1012`) and may send `SWITCH` (`1005`) to release the cores.
   - In response to `SWITCH`, the firmware finishes the current atomic work, saves context to session memory, sends `SWITCHED_OUT` (`core`, `reason`, `sub_reason`), and puts the core into `wait`.
4. **Buffer return protocol (`INPUT` / `OUTPUT`):**
   - For each buffer returned in `BUF_INRQ`, an `MVE_RESPONSE_CODE_INPUT` (`2005`, `size = 0`) notification must be enqueued in `MSG_OUTQ`.
   - For each buffer returned in `BUF_OUTRQ`, an `MVE_RESPONSE_CODE_OUTPUT` (`2006`, `size = 0`) notification must be enqueued in `MSG_OUTQ`.
   - The driver reads the corresponding buffer queue upon receiving `2005` or `2006` in `MSG_OUTQ`.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
