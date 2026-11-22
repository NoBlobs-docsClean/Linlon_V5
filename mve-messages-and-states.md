<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Protocolo MVE 3.5 — Mensagens de Controle, Eventos e Máquina de Estados

Este documento cataloga todas as mensagens de controle (requisições Host → Firmware e respostas Firmware → Host), eventos assíncronos, códigos de erro e a máquina de estados formal de uma sessão MVE 3.5 (`mve_protocol_def.h`, `[P][O]`).

---

## 1. Catálogo de Requisições (Host → Firmware em `MSG_INQ`) (`[P]`)

| Código | Macro Pública | Payload (`header.size`) | Semântica e Pré-Condições |
|---:|---|---|---|
| `1001` | `MVE_REQUEST_CODE_GO` | vazio (`0`) | Solicita transição para `RUNNING` (`2`). Rejeitar se já houver transição pendente ou se não houver job ativo na fila de jobs. Responde `STATE_CHANGE(RUNNING)`. |
| `1002` | `MVE_REQUEST_CODE_STOP` | vazio (`0`) | Solicita transição para `STOPPED` (`0`). Drena pipeline em curso e responde `STATE_CHANGE(STOPPED)`. Rejeitar se houver transição pendente. |
| `1003` | `MVE_REQUEST_CODE_INPUT_FLUSH` | vazio (`0`) | **Somente permitido em `STOPPED`.** Devolve todos os buffers pendentes da fila de entrada em `BUF_INRQ` (enviando `RESPONSE_CODE_INPUT` para cada um) e conclui com `INPUT_FLUSHED`. |
| `1004` | `MVE_REQUEST_CODE_OUTPUT_FLUSH` | vazio (`0`) | **Permitido em `STOPPED` ou no estado de espera após `SEQUENCE_PARAMETERS`.** Devolve todos os buffers de saída retidos em `BUF_OUTRQ` (com `RESPONSE_CODE_OUTPUT`) e conclui com `OUTPUT_FLUSHED`. |
| `1005` | `MVE_REQUEST_CODE_SWITCH` | vazio (`0`) | Solicita que o firmware salve o estado do LSID e libere os cores físicos; ao concluir, responde `SWITCHED_OUT` e entra em `wait`. |
| `1006` | `MVE_REQUEST_CODE_PING` | vazio (`0`) | Verificação de saúde (*health check*). Deve responder `MVE_RESPONSE_CODE_PONG` (`2009`). |
| `1008` | `MVE_REQUEST_CODE_DUMP` | vazio (`0`) | Solicita dump de estado interno para diagnóstico (ex.: `ram_print`). **Proibido em sessões protegidas** (deve retornar erro). Ao concluir, responde `MVE_RESPONSE_CODE_DUMP` (`2012`). |
| `1009` | `MVE_REQUEST_CODE_JOB` | `struct mve_request_job` (8 B) | Submete um job informando `cores`, `frames` e `flags`. Ao aceitar/desenfileirar em todos os cores da sessão, responde `JOB_DEQUEUED`. |
| `1010` | `MVE_REQUEST_CODE_SET_OPTION` | `struct mve_request_set_option` (variável) | **Somente permitido em `STOPPED`.** Configura opção de sessão (ver [mve-options.md](mve-options.md)). Responde `SET_OPTION_CONFIRM` ou `SET_OPTION_FAIL` (não-fatal). |
| `1011` | `MVE_REQUEST_CODE_RELEASE_REF_FRAME` | `struct mve_request_release_ref_frame` (4 B) | **Decode apenas.** Pede ao firmware que libere um buffer previamente retornado com `REF_FRAME` assim que não for mais usado como referência. Quando liberado, responde `REF_FRAME_UNUSED`. |
| `1012` | `MVE_REQUEST_CODE_IDLE_ACK` | vazio (`0`) | Confirmação do host (MVE >= 3.1) em resposta a um `MVE_RESPONSE_CODE_IDLE` enviado anteriormente pelo firmware. Erro se recebido sem `IDLE` pendente. |
| `1013` | `MVE_REQUEST_CODE_DEBUG` | `uint32_t level` (4 B) | Define nível de log do firmware: `0` = desabilitado, `1` = `FW_LOG_ERROR`, `2` = `WARNING`, `3` = `INFO`, `4` = `DEBUG`, `5` = `VERBOSE`. Responde `MVE_RESPONSE_CODE_DEBUG` (`2019`). |

### 1.1 Estruturas de Requisição

```c
struct mve_request_job {
    uint16_t cores;  /* >= 1; número de cores da sessão (deve bater com o escalonador HW) */
    uint16_t frames; /* número de frames a processar; 0 = infinito */
    uint32_t flags;  /* bit 0: MVE_JOB_FLAG_DISABLE_BNDMGR (0x01) */
};

struct mve_request_release_ref_frame {
    uint32_t buffer_address; /* VA do buffer de referência AFBC */
};
```

**Validações obrigatórias em `JOB` (`[P][D]`):** `cores` não pode ser `0` nem exceder o limite configurado/suportado (`1..8`); alteração dinâmica de `cores` entre jobs ativos não é suportada; flags desconhecidas devem ser rejeitadas; se não houver espaço na fila interna de jobs, reportar erro.

---

## 2. Catálogo de Respostas (Firmware → Host em `MSG_OUTQ`) (`[P]`)

| Código | Macro Pública | Payload (`header.size`) | Semântica |
|---:|---|---|---|
| `2001` | `MVE_RESPONSE_CODE_SWITCHED_IN` | `struct mve_response_switched_in` (4 B) | Enviada quando o firmware boota ou retoma execução em um core |
| `2002` | `MVE_RESPONSE_CODE_SWITCHED_OUT` | `struct mve_response_switched_out` (12 B) | Enviada quando o último core da sessão conclui o switch-out |
| `2003` | `MVE_RESPONSE_CODE_SET_OPTION_CONFIRM` | `struct mve_response_set_option_confirm` (4 B) | Confirma aplicação bem-sucedida da opção `index` |
| `2004` | `MVE_RESPONSE_CODE_JOB_DEQUEUED` | `struct mve_response_job_dequeued` (4 B) | Todos os cores da sessão desenfileiraram o job (`valid_job = 1` se válido) |
| `2005` | `MVE_RESPONSE_CODE_INPUT` | vazio (`0`) | Avisa que um buffer de entrada foi devolvido em `BUF_INRQ` (`0x1007_C000`) |
| `2006` | `MVE_RESPONSE_CODE_OUTPUT` | vazio (`0`) | Avisa que um buffer de saída foi devolvido em `BUF_OUTRQ` (`0x1007_E000`) |
| `2007` | `MVE_RESPONSE_CODE_INPUT_FLUSHED` | vazio (`0`) | Conclui `INPUT_FLUSH` |
| `2008` | `MVE_RESPONSE_CODE_OUTPUT_FLUSHED` | vazio (`0`) | Conclui `OUTPUT_FLUSH` |
| `2009` | `MVE_RESPONSE_CODE_PONG` | vazio (`0`) | Resposta a `PING` |
| `2010` | `MVE_RESPONSE_CODE_ERROR` | `struct mve_response_error` (132 B) | **Erro fatal da sessão** (nenhuma operação adicional é garantida após envio) |
| `2011` | `MVE_RESPONSE_CODE_STATE_CHANGE` | `struct mve_response_state_change` (4 B) | Confirma mudança de estado (`0 = STOPPED`, `2 = RUNNING`) |
| `2012` | `MVE_RESPONSE_CODE_DUMP` | vazio (`0`) | Conclui `DUMP` |
| `2013` | `MVE_RESPONSE_CODE_IDLE` | vazio (`0`) | Informa ao host que não há trabalho executável (aguardando buffers ou idle) |
| `2014` | `MVE_RESPONSE_CODE_FRAME_ALLOC_PARAM` | `struct mve_response_frame_alloc_parameters` (32 B) | **Decode apenas.** Informa requisitos de tamanho de buffers de saída |
| `2015` | `MVE_RESPONSE_CODE_SEQUENCE_PARAMETERS` | `struct mve_response_sequence_parameters` (8 B) | **Decode apenas.** Informa parâmetros de sequência; entra em pausa até `OUTPUT_FLUSH` |
| `2016` | `MVE_RESPONSE_CODE_EVENT` | `struct mve_response_event` (variável) | Evento informativo/diagnóstico (não exige resposta obrigatória do host) |
| `2017` | `MVE_RESPONSE_CODE_SET_OPTION_FAIL` | `struct mve_response_set_option_fail` (132 B) | **Erro não-fatal** ao configurar opção `index` (sessão continua viva) |
| `2018` | `MVE_RESPONSE_CODE_REF_FRAME_UNUSED` | `struct mve_response_ref_frame_unused` (4 B) | Confirma que `unused_buffer_address` não é mais referência |
| `2019` | `MVE_RESPONSE_CODE_DEBUG` | vazio (`0`) | Confirma aplicação de `MVE_REQUEST_CODE_DEBUG` |

### 2.1 Estruturas de Resposta

```c
struct mve_response_switched_in  { uint32_t core; };
struct mve_response_switched_out { uint32_t core; uint32_t reason; uint32_t sub_reason; };
struct mve_response_state_change { uint32_t new_state; }; /* 0=MVE_STATE_STOPPED, 2=MVE_STATE_RUNNING */
struct mve_response_job_dequeued { uint32_t valid_job; };
struct mve_response_set_option_confirm { uint32_t index; };
struct mve_response_ref_frame_unused   { uint32_t unused_buffer_address; };

/* Nota [P][O]: Em fw_v3/mve_protocol_def.h o comentário ao lado da macro
 * MVE_RESPONSE_CODE_SET_OPTION_CONFIRM (2003) indica "no extra data",
 * porém o próprio header define struct mve_response_set_option_confirm
 * e os traços reais em hardware (K1/K3) transmitem sempre os 4 bytes de index
 * (header.size = 4). */

struct mve_response_error {
    uint32_t error_code;
    char     message[128]; /* string ASCII terminada em '\0' */
};

struct mve_response_set_option_fail {
    uint32_t index;
    char     message[128];
};
```

#### Códigos de Erro (`mve_response_error.error_code`)

| Código | Macro Pública | Causa |
|---:|---|---|
| `1` | `MVE_ERROR_ABORT` | Exceção fatal de CPU RASC ou `MMU ABORT` |
| `2` | `MVE_ERROR_OUT_OF_MEMORY` | Falha de alocação (RPC retornou 0 ou `IntBuf` insuficiente) |
| `3` | `MVE_ERROR_ASSERT` | Falha de invariante interna (assert) |
| `4` | `MVE_ERROR_UNSUPPORTED` | Código de mensagem ou funcionalidade não suportada |
| `6` | `MVE_ERROR_INVALID_BUFFER` | Descritor de buffer inválido ou fora das regiões permitidas |
| `8` | `MVE_ERROR_INVALID_STATE` | Comando inválido para o estado atual (ex.: `INPUT_FLUSH` em `RUNNING`) |
| `9` | `MVE_ERROR_WATCHDOG` | Estouro do temporizador de watchdog interno |

### 2.2 Parâmetros de Alocação e Sequência (Decode)

```c
struct mve_response_frame_alloc_parameters {
    uint16_t planar_alloc_frame_width;             /* +0x00 */
    uint16_t planar_alloc_frame_height;            /* +0x02 */
    uint32_t afbc_alloc_bytes;                     /* +0x04 (por campo se interlace) */
    uint32_t afbc_alloc_bytes_downscaled;          /* +0x08 */
    uint16_t afbc_width_in_superblocks;            /* +0x0C */
    uint16_t afbc_width_in_superblocks_downscaled; /* +0x0E */
    uint16_t cropx;                                /* +0x10: offset X p/ bursts AXI ideais */
    uint16_t cropy;                                /* +0x12: offset Y p/ bursts AXI ideais */
    uint32_t mbinfo_alloc_bytes;                   /* +0x14 (debug) */
    uint16_t dsl_frame_width;                      /* +0x18 (arbitrary downscale) */
    uint16_t dsl_frame_height;                     /* +0x1A */
    uint16_t dsl_pos_mode;                         /* +0x1C */
    uint8_t  ctu_size;                             /* +0x1E (16 para H.264, 16/32/64 para HEVC) */
    /* +0x1F: 1 byte padding (sizeof = 32) */
};

struct mve_response_sequence_parameters {
    uint8_t interlace;             /* +0x00: 0=progressivo, 1=entrelaçado */
    uint8_t chroma_format;         /* +0x01: 0=MONO, 1=420, 2=422, 3=440, 4=ARGB */
    uint8_t bitdepth_luma;         /* +0x02: 8, 9 ou 10 */
    uint8_t bitdepth_chroma;       /* +0x03: 8, 9 ou 10 */
    uint8_t num_buffers_planar;    /* +0x04: mínimo de buffers planar */
    uint8_t num_buffers_afbc;      /* +0x05: mínimo de buffers AFBC (inclui DPB/refs) */
    uint8_t range_mapping_enabled; /* +0x06: específico de VC-1 AP */
    uint8_t reserved0;             /* +0x07 */
};
```

---

## 3. Eventos (`MVE_RESPONSE_CODE_EVENT` = `2016`) (`[P]`)

`struct mve_response_event` é empacotada (`__attribute__((packed))`): `uint32_t event_code` seguido da união `event_data`:

| `event_code` | Macro Pública | Payload em `event_data` |
|---:|---|---|
| `1` | `MVE_EVENT_ERROR_STREAM_CORRUPT` | `char message[128]` — corrupção recuperável no bitstream |
| `2` | `MVE_EVENT_ERROR_STREAM_NOT_SUPPORTED` | `char message[128]` — recurso do stream não suportado |
| `3` | `MVE_EVENT_PROCESSED` | `struct mve_event_processed` (84 B) — telemetria de frame (se `PROFILING=1`) |
| `4` | `MVE_EVENT_REF_FRAME` | `struct mve_event_ref_frame` (40 B) — metadados de referência AFBC (se `ENC_EXPOSE_REF_FRAMES=1`) |
| `5` | `MVE_EVENT_TRACE_BUFFERS` | `struct mve_event_trace_buffers` — endereços RASC/tamanhos de até 40 buffers de trace |

### 3.1 `struct mve_event_processed` (84 Bytes)

| Offset | Tipo | Campo | Significado |
|---:|---|---|---|
| `0x00` | `uint8_t` | `pic_format`, `qp`, `pad0`, `pad1` | Formato do quadro e QP médio |
| `0x04` | `uint32_t` | `parse_start_time`, `parse_end_time`, `parse_idle_time` | Timestamps e ociosidade do estágio de *parse* |
| `0x10` | `uint32_t` | `pipe_start_time`, `pipe_end_time`, `pipe_idle_time` | Timestamps e ociosidade do estágio de *pipe* (`pipe_idle_time = 0` em decode) |
| `0x1C` | `uint32_t` | `parser_coreid`, `pipe_coreid` | IDs dos cores que executaram *parse* e *pipe* |
| `0x24` | `uint32_t` | `bitstream_bits` | Bits de bitstream consumidos/produzidos no frame |
| `0x28` | `uint32_t` | `intermediate_buffer_size`, `total_memory_allocated` | Uso de `IntBuf` e memória total alocada (incluindo refs) |
| `0x30` | `uint32_t` | `bus_read_bytes`, `bus_write_bytes`, `afbc_bytes` | Contadores de tráfego AXI e AFBC |
| `0x3C` | `uint32_t` | `slice0_end_time`, `stream_start_time`, `stream_open_time` | Timestamps de latência |

---

## 4. Máquina de Estados da Sessão (`[P][O][D]`)

```text
                       ┌────────────────────────┐
       [Boot/Init] ───►│     STOPPED (0)        │◄───────────────────────┐
                       │ • Aceita SET_OPTION    │                        │
                       │ • Aceita INPUT_FLUSH   │                        │
                       │ • Aceita OUTPUT_FLUSH  │                        │
                       │ • Aceita JOB / PING    │                        │
                       └───────────┬────────────┘                        │
                                   │ GO (1001)                           │ STOP (1002)
                                   │ → STATE_CHANGE(2)                   │ → STATE_CHANGE(0)
                                   ▼                                     │
                       ┌────────────────────────┐                        │
                       │     RUNNING (2)        │────────────────────────┘
                       │ • Processa IN/OUT bufs │
                       │ • Rejeita SET_OPTION   │
                       │ • Rejeita INPUT_FLUSH  │
                       │ • Emite IDLE se vazio  │
                       └───────────┬────────────┘
                                   │ (Decode: mudança de sequência)
                                   │ → FRAME_ALLOC_PARAM + SEQUENCE_PARAMETERS
                                   ▼
                       ┌────────────────────────┐
                       │   WAIT_OUTPUT_FLUSH    │
                       │ • Pausa produção       │
                       │ • Aguarda OUTPUT_FLUSH │
                       │   → devolve OUT bufs   │
                       │   → OUTPUT_FLUSHED     │
                       └───────────┬────────────┘
                                   │ (Retoma após OUTPUT_FLUSHED e novos OUT bufs)
                                   └──────────────► RUNNING (2)
```

### Invariantes Obrigatórias da Máquina de Estados

1. **Transição única por vez:** Se `GO` ou `STOP` chegar enquanto uma transição anterior ainda estiver pendente, responder erro (`MVE_ERROR_INVALID_STATE`).
2. **Restrições por estado:**
   - `SET_OPTION` fora de `STOPPED` → `MVE_RESPONSE_CODE_SET_OPTION_FAIL`.
   - `INPUT_FLUSH` fora de `STOPPED` → `MVE_ERROR_INVALID_STATE`.
   - `OUTPUT_FLUSH` só é aceito em `STOPPED` ou em `WAIT_OUTPUT_FLUSH` (após `SEQUENCE_PARAMETERS`).
3. **Interação com `IDLE` e `SWITCH`:**
   - Em `RUNNING`, quando não há buffers ou trabalho realizável, o firmware envia `IDLE` (`2013`).
   - O host (MVE >= 3.1) responde `IDLE_ACK` (`1012`) e pode enviar `SWITCH` (`1005`) para liberar os cores.
   - Em resposta a `SWITCH`, o firmware finaliza o trabalho atômico corrente, salva o contexto na memória da sessão, envia `SWITCHED_OUT` (`core`, `reason`, `sub_reason`) e coloca o core em `wait`.
4. **Protocolo de devolução de buffer (`INPUT` / `OUTPUT`):**
   - Para cada buffer devolvido em `BUF_INRQ`, deve ser enfileirada uma notificação `MVE_RESPONSE_CODE_INPUT` (`2005`, `size = 0`) em `MSG_OUTQ`.
   - Para cada buffer devolvido em `BUF_OUTRQ`, deve ser enfileirada uma notificação `MVE_RESPONSE_CODE_OUTPUT` (`2006`, `size = 0`) em `MSG_OUTQ`.
   - O driver lê a fila de buffers correspondente ao receber `2005` ou `2006` em `MSG_OUTQ`.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
