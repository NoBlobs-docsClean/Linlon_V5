<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Watchdog, Detecção de Falhas e Recuperação de Hardware (`hardware-recovery`)

Este documento especifica os mecanismos de detecção, diagnóstico e recuperação de falhas de hardware na VPU Linlon V5 (`0x56635002`): temporizador de watchdog interno (`0xFF02_5000`), falhas de MMU por bus master (`0xFF02_4000`), exceções de CPU RASC (`CR1`/`CR2`/`CR18`/`SR`), falhas de SDMA (`0xFF02_1F00`), comando `DUMP` e o protocolo de encerramento forçado pelo host (`TERMINATE`).

---

## 1. Watchdog de Hardware do Core (`0xFF02_5000`) (`[P][D]`)

O core possui um bloco temporizador de watchdog dedicado em **`0xFF02_5000`**, configurado conforme a opção MVE `MVE_SET_OPT_INDEX_WATCHDOG_TIMEOUT` (`42`, onde `0` = desabilitado):

| Endereço | Offset | Largura | Acesso | Função |
|---|---:|---:|---|---|
| **`0xFF02_5000`** | `+0x00` | 32b | W | **Comparador / Deadline de Watchdog:** dispara IRQ bit 4 (`0x10`) quando o contador `+0x10` atinge este valor |
| **`0xFF02_5010`** | `+0x10` | 32b | R | **Contador livre (Free-Running Timer):** base de tempo monotônica do watchdog |
| **`0xFF02_5014`** | `+0x14` | 32b | W | **Limpeza de IRQ do Watchdog:** escrever **`1`** limpa a interrupção pendente |

### 1.1 Operação Normativa do Watchdog (`[D]`)

- **Armar / Rearmar (Kick):**
  ```c
  uint32_t now = *(volatile uint32_t *)0xFF025010u;
  *(volatile uint32_t *)0xFF025000u = now + timeout_ticks;
  ```
  - Habilitar bit 4 (`0x10`) no controlador de IRQ da RASC supervisora (`0xFF02_6000 + rasc_id * 0x1000`).
  - Rearmar a cada progresso de frame/stripe ou no tratamento de eventos de trabalho.
- **Ao disparar IRQ de Watchdog (bit 4 = `0x10` em `IRQ_STATUS`):**
  1. Escrever **`1`** em `0xFF02_5014` para limpar a interrupção de hardware.
  2. Opcionalmente registrar diagnóstico (estado dos blocos e contexto RASC) em `ram_print`.
  3. Enviar **`MVE_RESPONSE_CODE_ERROR`** (`2010`) com **`error_code = MVE_ERROR_WATCHDOG` (`9`)** e mensagem descritiva em `MSG_OUTQ`, seguido de `CR14 = 0` e `0xFF02B028 = 1`.
  4. Entrar em estado seguro/parado aguardando `TERMINATE` do host.

> **Watchdog do lado Host (`[P][O]`):** Independentemente do watchdog interno do firmware, o driver Linux mantém um watchdog de sessão próprio (default observado: `30.000 ms`). Se o firmware travar sem responder, o driver invoca `mvx_lsid_terminate` e sinaliza erro à camada V4L2.

---

## 2. Tratamento de `MMU ABORT` — Controlador de 8 Bus Masters (`0xFF02_4000`) (`[D]`)

Quando qualquer unidade interna tenta acessar um endereço virtual não mapeado (`AP = 0`) ou viola permissões de página (`RO`/`X`/`RW`), o hardware dispara o **bit 3 (`0x0000_0008`)** em `IRQ_STATUS` do controlador de interrupção da RASC supervisora.

O bloco MMU interno em **`0xFF02_4000`** mantém pares de registradores `FAULT` / `FADDR` para **8 bus masters**, com stride de **`0x20` bytes** por master (`m = 0..7`):

```text
MMU_FAULT(m) = 0xFF02_4010 + m * 0x20    (32b R/W: != 0 indica falha ativa; escrever 0 limpa)
MMU_FADDR(m) = 0xFF02_4014 + m * 0x20    (32b R:   endereço virtual de 32 bits que causou a falha)
```

| Master `m` | Subsistema Bus Master | `MMU_FAULT` (R/W0C) | `MMU_FADDR` (R) |
|---:|---|---|---|
| `0` | **RASC0** | `0xFF02_4010` | `0xFF02_4014` |
| `1` | **RASC1** | `0xFF02_4030` | `0xFF02_4034` |
| `2` | **RASC2** | `0xFF02_4050` | `0xFF02_4054` |
| `3` | **RASC3** | `0xFF02_4070` | `0xFF02_4074` |
| `4` | **RASC4** | `0xFF02_4090` | `0xFF02_4094` |
| `5` | **SDMA** | `0xFF02_40B0` | `0xFF02_40B4` |
| `6` | **VDMA** | `0xFF02_40D0` | `0xFF02_40D4` |
| `7` | **REF** | `0xFF02_40F0` | `0xFF02_40F4` |

### Sequência Obrigatória de Tratamento de `MMU ABORT`

1. Iterar `m = 0 .. 7`:
   - Ler `fault = *(volatile uint32_t *)(0xFF024010 + m * 0x20)`.
   - Se `fault != 0`:
     - Ler `faddr = *(volatile uint32_t *)(0xFF024014 + m * 0x20)`.
     - Registrar master `m`, `fault` e `faddr` em `ram_print` / mensagem de erro.
     - Limpar a falha escrevendo **`0`** em `0xFF024010 + m * 0x20`.
2. Enviar **`MVE_RESPONSE_CODE_ERROR`** (`2010`) com **`error_code = MVE_ERROR_ABORT` (`1`)** ao host (`CR14 = 0` + `0xFF02B028 = 1`).
3. Cessar operações de vídeo na sessão (erro fatal) e aguardar `TERMINATE` do host.

---

## 3. Exceções de CPU (`RASC ABORT`) (`[D]`)

Quando uma exceção síncrona atinge as entradas `1..3` da tabela de vetores em `CR0`:

1. O stub salva `r0..r14` na pilha (`push r0-r14`), lê:
   - **`CR2`** (`mfcr r, 2`) = `FAR` (*Fault Address Register*),
   - **`CR1`** (`mfcr r, 1`) = `PC` da instrução faltosa,
   - **`SR`** (`mfsr r`) = *Status Register*,
   - **`CR18`** (`mfcr r, 0x12`) = `RASC_ID` (`0..4`),
   e empilha `(FAR, PC, SR)` no topo do frame.
2. O tratador registra `RASC_ID`, `PC`, `SR`, `FAR` e `R[0..14]` (e opcionalmente faz dump da pilha e dos 8 bus masters da MMU em `0xFF02_4010+`).
3. Envia `MVE_RESPONSE_CODE_ERROR` (`MVE_ERROR_ABORT = 1`) se o canal MVE estiver utilizável e entra em parada segura (nunca executa `rte` em abort fatal).

---

## 4. Falhas de DMA e Blocos Internos (`[D]`)

- **Falha de SDMA:** Ler `*(volatile uint32_t *)0xFF021F00` (registrador de falha do controlador SDMA). Valor diferente de zero indica erro de transferência em canal SDMA; detalhes de canais podem ser inspecionados em `0xFF02_1000` (status), `0xFF02_1600` (dispositivos pendentes) e `0xFF02_1804`/`0xFF02_180C` (descritores).
- **Erros recuperáveis de bitstream (não-fatais):** Em decodificação, cabeçalhos ou slices corrompidos **não** devem abortar a sessão com `MVE_RESPONSE_CODE_ERROR`; em vez disso:
  - Emitir `MVE_RESPONSE_CODE_EVENT` (`2016`) com `MVE_EVENT_ERROR_STREAM_CORRUPT` (`1`) ou `MVE_EVENT_ERROR_STREAM_NOT_SUPPORTED` (`2`).
  - Marcar eventuais quadros afetados com `MVE_BUFFER_FRAME_FLAG_CORRUPT` (`0x2000`) ao devolvê-los em `BUF_OUTRQ`.

---

## 5. Comando `MVE_REQUEST_CODE_DUMP` (`1008`) (`[P][D]`)

Quando o host envia `MVE_REQUEST_CODE_DUMP`:
1. **Sessão protegida:** Se a sessão for protegida, **proibir** o dump e reportar erro (`MVE_ERROR_UNSUPPORTED` / `MVE_ERROR_INVALID_STATE`), para não vazar metadados de conteúdo protegido.
2. **Sessão normal:** Gravar snapshot de diagnóstico em `ram_print` (`0x1010_0000`), cobrindo:
   - Revisão interna do silício (`0xFF02_B06C`) e relógio do core (`0xFF02_B004`),
   - Estado lógico da sessão e filas MVE,
   - Registradores de status dos blocos de vídeo (`BITSTREAM`, `REF`, `MC`, `TRF`, `DBL`, `SDMA`, `VDMA`).
3. Enviar **`MVE_RESPONSE_CODE_DUMP`** (`2012`, `size = 0`) ao host.

---

## 6. Recuperação Forçada pelo Host (`TERMINATE` e `RESET`) (`[P]`)

Caso uma sessão não responda após erro fatal ou timeout:
1. O host remove o LSID de `JOBQUEUE` (com `ENABLE = 0` → atualiza `JOBQUEUE` → `ENABLE = 1`).
2. O host escreve `1` em `LSID.TERMINATE` (`+0x18`) e faz poll até `TERMINATE == 0` (o hardware aborta imediatamente todos os cores que estejam executando aquele LSID e libera suas entradas em `CORELSID`).
3. O host desmapeia o LSID (`SCHED = 0`, `ALLOC = 0`) e libera as tabelas MMU.
4. Em caso de suspensão/retomada ou travamento global do bloco, o host executa `reset_hw()`: `RESET (0x50) = 1` → poll `CORELSID == corelsid_mask` (`0xFF` no K1, `0xFFFF` no K3) → `CLKFORCE (0x24) = 0`.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
