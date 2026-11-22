<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Interrupções RASC, Doorbells e Sincronização Multi-Core (`rasc-irq-and-sync`)

Este documento especifica o modelo de interrupções das 5 CPUs RASC por core, a tabela de vetores configurada em `CR0`, os 5 controladores de interrupção dedicados (`0xFF02_6000`–`0xFF02_A000`), a entrega e limpeza de `IRQHOST` (Host → Firmware), a sinalização inter-RASC (`0xFF02_3000`), os doorbells inter-core (`0xFF02_B040+`) e o spinlock de hardware (`0xFF02_B038`).

---

## 1. Tabela de Vetores de Exceção e Interrupção (`CR0` — VBAR) (`[D]`)

Cada CPU RASC possui um registrador de base da tabela de vetores em **`CR0`**. O ponteiro gravado em `CR0` (carregado via `lpc`) aponta para um vetor contíguo de **4 endereços virtuais de 32 bits** (16 bytes):

| Índice | Offset | Evento de Hardware | Retorno Esperado |
|---:|---:|---|---|
| `0` | `+0x00` | **IRQ Externa** (sinalizada pelo controlador `0xFF02_6000 + rasc_id * 0x1000`) | Salva contexto (`push r0-r14`, `CR2`/`CR1`/`SR`), trata IRQ, restaura e executa **`rte`** |
| `1` | `+0x04` | **Exceção / Abort 1** (ex.: abort de instrução / acesso ilegal) | Salva contexto, reporta `RASC ABORT` (`PC`/`SR`/`FAR`/`R0..R14`) e para |
| `2` | `+0x08` | **Exceção / Abort 2** (ex.: abort de dados / MMU) | Salva contexto, reporta `RASC ABORT` e para |
| `3` | `+0x0C` | **Exceção / Abort 3** (ex.: exceção reservada / erro fatal) | Salva contexto, reporta `RASC ABORT` e para |

> **Observação de endereçamento (`[D]`):** As 4 entradas da tabela são endereços virtuais de 32 bits (base `0x1000 + offset_arquivo` na instância 0). O hardware despacha indiretamente pela entrada correspondente quando ocorre IRQ ou exceção.

---

## 2. Controle Local de Interrupções na CPU (`clri` / `seti`) (`[P][D]`)

As instruções `clri` e `seti` aceitam um registrador escalar (`clri rN` / `seti rN`):

| Instrução | Semântica de Hardware |
|---|---|
| **`clri rN`** | Salva o estado anterior de habilitação de interrupções em **`rN`** e **desabilita** interrupções na CPU RASC atual |
| **`seti rN`** | Restaura o estado de habilitação de interrupções a partir de **`rN`** (seção crítica aninhável segura!) |

**Padrão normativo de seção crítica:**
```asm
clri r3          ; salva estado anterior em r3 e mascara IRQs
; ... seção crítica ...
seti r3          ; restaura estado anterior a partir de r3
```

---

## 3. Controladores de Interrupção por RASC (`0xFF02_6000` – `0xFF02_A000`) (`[D]`)

Cada uma das 5 CPUs RASC dentro de um core possui seu próprio banco de controlador de interrupção, espaçado de **`0x1000` (4 KiB)**:

```text
IRQ_CTRL_BASE(rasc_id) = 0xFF02_6000 + rasc_id * 0x1000    (rasc_id = 0..4)
```

| CPU RASC | Base MMIO do Controlador de IRQ |
|---:|---|
| **RASC0** | `0xFF02_6000` |
| **RASC1** | `0xFF02_7000` |
| **RASC2** | `0xFF02_8000` |
| **RASC3** | `0xFF02_9000` |
| **RASC4** | `0xFF02_A000` |

### 3.1 Registradores de Cada Banco (`+0x00`, `+0x04`, `+0x08`)

| Offset | Largura | Acesso | Nome Lógico | Função |
|---:|---:|---|---|---|
| `+0x00` | 32b | R | `IRQ_STATUS` | Máscara de bits das interrupções pendentes para esta RASC |
| `+0x04` | 32b | R/W | `IRQ_ENABLE` | Máscara primária de habilitação de IRQs (bit `1` = habilitada) |
| `+0x08` | 32b | R/W | `IRQ_MASK2` / `IRQ_MODE` | Máscara secundária / configuração de modo das linhas de IRQ |

### 3.2 Mapa de Bits de `IRQ_STATUS` (`+0x00`) e `IRQ_ENABLE` (`+0x04`)

Os bits confirmados no controlador de interrupção das CPUs RASC são:

| Bit | Máscara Hex | Fonte de Hardware | Como Limpar / Reconhecer a Origem |
|---:|---|---|---|
| `1` | `0x0000_0002` | **Doorbell Inter-RASC** (sinal enviado por outra CPU RASC no mesmo core) | Escrever **`0`** em `0xFF02_3040 + rasc_id * 4` e/ou `0xFF02_3060 + rasc_id * 4` |
| `3` | `0x0000_0008` | **`MMU ABORT`** (falha de tradução/permissão em qualquer um dos 8 bus masters) | Ler `0xFF02_4010 + m * 0x20` (`m = 0..7`), ler o endereço faltoso em `+0x14` e escrever **`0`** em `+0x10` (ver [hardware-recovery.md](hardware-recovery.md)) |
| `4` | `0x0000_0010` | **`WATCHDOG` Timer IRQ** (comparador `0xFF02_5000` atingido) | Escrever **`1`** em **`0xFF02_5014`** |
| `17` | `0x0002_0000` | **Evento Interno de Job / Subsistema** | Tratado conforme estado da fila de jobs / sinalização interna |
| **`18`** | **`0x0004_0000`** | **`IRQHOST` (Host → Firmware)** — disparada quando o driver Linux escreve `1` em `LSID.IRQHOST` | Escrever **`0`** em **`0xFF02_B02C`** |
| **`23`** | **`0x0080_0000`** | **Doorbell Inter-Core** — disparada quando outro core físico escreve `1` em `0xFF02_B040 + core_id * 4` | Escrever **`0`** em **`0xFF02_B040 + core_id * 4`** (onde `core_id = [0xFF02_B000]`) |

> **Configuração das máscaras por RASC (`[D]`):** A escolha de quais linhas de IRQ habilitar em `IRQ_ENABLE` (`+0x04`) e `IRQ_MASK2` (`+0x08`) de cada CPU RASC depende exclusivamente da arquitetura escolhida pelo firmware clean-room. Por exemplo, a RASC designada como supervisora MVE habilita os bits correspondentes a `MMU ABORT` (bit 3), `WATCHDOG` (bit 4), `IRQHOST` (bit 18) e Inter-Core (bit 23), enquanto CPUs auxiliares habilitam o doorbell Inter-RASC (bit 1) e as linhas dos aceleradores que gerenciam.

---

## 4. Sinalização Host ↔ Firmware (`0xFF02_B028` e `0xFF02_B02C`) (`[O][D]`)

| Direção | Registrador MMIO | Operação | Efeito |
|---|---|---|---|
| **Firmware → Host** | **`0xFF02_B028`** (`[O]`) | `CR14 = 0`, depois escrever **`1`** | Seta o bit do LSID em `IRQVE` (`0xC050_0018`) e dispara IRQ 74 para o Linux |
| **Host → Firmware (recepção)** | **`0xFF02_6000 + rasc_id * 0x1000`** (banco da RASC que atende o host) (`[D]`) | Testar **bit 18 (`0x0004_0000`)** de `IRQ_STATUS` | Indica que o driver escreveu `1` em `LSID.IRQHOST` (novas mensagens/buffers ou resposta RPC) |
| **Host → Firmware (limpeza)** | **`0xFF02_B02C`** (`[D]`) | Escrever **`0`** | Limpa a pendência de `IRQHOST` no hardware do core |

---

## 5. Doorbells Inter-RASC (`0xFF02_3040..0xFF02_3070`) e Inter-Core (`0xFF02_B040+`) (`[D]`)

### 5.1 Inter-RASC (Dentro do Mesmo Core — Bloco `0xFF02_3000`)

| Canal | RASC0 | RASC1 | RASC2 | RASC3 | RASC4 | Uso |
|---|---|---|---|---|---|---|
| **Doorbell / Sinal A** | `0xFF02_3040` | `0xFF02_3044` | `0xFF02_3048` | `0xFF02_304C` | `0xFF02_3050` | Escrever `1` para sinalizar (ou `0` para limpar) a RASC destino |
| **Doorbell / Sinal B** | `0xFF02_3060` | `0xFF02_3064` | `0xFF02_3068` | `0xFF02_306C` | `0xFF02_3070` | Acorda a RASC destino (bit 1 em `IRQ_STATUS`); escrever `0` no tratador para limpar |

### 5.2 Inter-Core (Entre Cores Físicos do Mesmo LSID — Bloco `0xFF02_B000`)

- **Identificação do core atual:** `core_id = *(volatile uint32_t *)0xFF02B000` (`0..1` no K1; `0..3` no K3).
- **Identificação da RASC atual:** `rasc_id = mfcr(0x12)` (`0..4`).
- **Enviar IRQ para o core `target_core`:**
  1. Executar **`CR14 = 0`** para garantir visibilidade da Session SRAM (`0xFF00_0000`) e `master_rw`.
  2. Escrever **`1`** em **`*(volatile uint32_t *)(0xFF02B040 + target_core * 4)`**.
- **Limpar IRQ inter-core recebida (bit 23 = `0x0080_0000` em `IRQ_STATUS`):**
  - Escrever **`0`** em **`*(volatile uint32_t *)(0xFF02B040 + my_core_id * 4)`**.

---

## 6. Spinlock / Mutex de Hardware (`0xFF02_B038`) (`[D]`)

O registrador **`0xFF02_B038`** implementa um **mutex atômico de hardware** compartilhado por todos os cores físicos e todas as CPUs RASC de uma sessão:

- **Regra de hardware:** Uma escrita não-zero em `0xFF02_B038` só é aceita se o registrador estiver atualmente em **`0`** (livre). Se já estiver ocupado por outro dono, a escrita é ignorada e a leitura subsequente retorna o token do dono atual. Escrever **`0`** libera o lock.
- **Formato do token único por `(core_id, rasc_id)`:**
  ```c
  uint32_t core_id = *(volatile uint32_t *)0xFF02B000u;
  uint32_t rasc_id = rasc_mfcr_0x12(); /* CR18: 0..4 */
  uint32_t token   = ((core_id + 1u) << 4) | (rasc_id & 0xFu);
  /* Nota: core_id + 1 garante que token é SEMPRE != 0! */
  ```
- **Sequência de aquisição (reentrante segura):**
  1. Se `*(volatile uint32_t *)0xFF02B038 == token`, esta CPU já detém o lock (retorna 0 / já adquirido).
  2. Desabilitar interrupções locais (`clri saved`).
  3. Em laço:
     - Escrever `token` em `0xFF02B038`.
     - Ler `0xFF02B038` de volta.
     - Repetir enquanto o valor lido `!= token`.
  4. Restaurar interrupções (`seti saved`) ou manter desabilitadas durante a seção crítica.
- **Sequência de liberação:**
  1. Executar **`CR14 = 0`** para garantir que todas as escritas na seção crítica sejam visíveis na Session SRAM / DRAM antes de soltar o lock.
  2. Escrever **`0`** em **`*(volatile uint32_t *)0xFF02B038`**.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
