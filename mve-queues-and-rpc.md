<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Protocolo MVE 3.5 — Filas Circulares, RPC e RAM de Log

Este documento especifica a camada de transporte em memória compartilhada do protocolo **MVE 3.5** (`mve_protocol_def.h` e `mvx_firmware_v2.c`, `[P]`): os 6 anéis circulares de 4 KiB, as regras de empacotamento e *wrap-around*, o canal síncrono de RPC e o buffer circular de log em RAM (`ram_print`).

---

## 1. Mapa de Endereços Virtuais dos Canais (`[P]`)

Todos os canais abaixo são mapeados pelo driver host com `MVX_ATTR_SHARED_RW` e `MVX_ACCESS_READ_WRITE` em endereços virtuais fixos idênticos para todos os cores da sessão:

| Canal | VA Fixo | Tamanho | Estrutura | Escritor Principal |
|---|---|---:|---|---|
| `MVE_COMM_MSG_INQ_ADDR` | `0x1007_9000` | 4 KiB | `struct mve_comm_area_host` | Host (requisições → FW) |
| `MVE_COMM_MSG_OUTQ_ADDR` | `0x1007_A000` | 4 KiB | `struct mve_comm_area_mve` | Firmware (respostas → Host) |
| `MVE_COMM_BUF_INQ_ADDR` | `0x1007_B000` | 4 KiB | `struct mve_comm_area_host` | Host (buffers de entrada → FW) |
| `MVE_COMM_BUF_INRQ_ADDR` | `0x1007_C000` | 4 KiB | `struct mve_comm_area_mve` | Firmware (buffers de entrada devolvidos → Host) |
| `MVE_COMM_BUF_OUTQ_ADDR` | `0x1007_D000` | 4 KiB | `struct mve_comm_area_host` | Host (buffers de saída → FW) |
| `MVE_COMM_BUF_OUTRQ_ADDR` | `0x1007_E000` | 4 KiB | `struct mve_comm_area_mve` | Firmware (buffers de saída devolvidos → Host) |
| `MVE_COMM_RPC_ADDR` | `0x1007_F000` | 4 KiB | `struct mve_rpc_communication_area` | Bidirecional (handshake por `state`) |
| `MVE_FW_PRINT_RAM_ADDR` | `0x1010_0000` | 512 KiB | `struct mve_fw_ram_print_head_aera` + slots | Firmware escreve `wr_cnt`/slots; Host atualiza `rd_cnt` |

---

## 2. Layout das Filas Circulares (`[P]`)

Cada par de canais (Mensagens, Buffers de Entrada, Buffers de Saída) é formado por **duas páginas de 4 KiB** — uma página escrita exclusivamente pelo Host e outra escrita exclusivamente pelo Firmware — eliminando *false sharing* e conflitos de escrita concorrente:

```c
#define MVE_COMM_QUEUE_SIZE_IN_WORDS 1020

/* Página escrita APENAS pelo Host (0x10079000, 0x1007B000, 0x1007D000) */
struct mve_comm_area_host {
    volatile uint16_t out_rpos;      /* offset 0x00: índice em words que o Host já leu de mve->out_data */
    volatile uint16_t in_wpos;       /* offset 0x02: índice em words até onde o Host escreveu em host->in_data */
    volatile uint32_t reserved[3];   /* offset 0x04..0x0F: reservado (12 bytes) */
    volatile uint32_t in_data[1020]; /* offset 0x10..0xFFF: anel circular de 1020 words (4080 bytes) */
};

/* Página escrita APENAS pelo Firmware (0x1007A000, 0x1007C000, 0x1007E000) */
struct mve_comm_area_mve {
    volatile uint16_t out_wpos;      /* offset 0x00: índice em words até onde o FW escreveu em mve->out_data */
    volatile uint16_t in_rpos;       /* offset 0x02: índice em words que o FW já leu de host->in_data */
    volatile uint32_t reserved[3];   /* offset 0x04..0x0F: reservado (12 bytes) */
    volatile uint32_t out_data[1020];/* offset 0x10..0xFFF: anel circular de 1020 words (4080 bytes) */
};
```

### 2.1 Cabeçalho de Mensagem (`struct mve_msg_header`, 4 Bytes)

Toda entrada em qualquer uma das 6 filas começa por uma word de 32 bits contendo:

```c
struct mve_msg_header {
    uint16_t code; /* bits [15:0]: código da mensagem (1001..1013, 2001..2019, 3001..3004) */
    uint16_t size; /* bits [31:16]: tamanho em BYTES do payload que segue o cabeçalho (0 se vazio) */
};
```

> **Regra de versionamento (`[P]`):** O consumidor **sempre** deve consumir `size` bytes (arredondados para cima em words de 32 bits) com base em `header.size`, nunca assumindo `sizeof(struct)` fixo, para tolerar expansões futuras de payload.

### 2.2 Regras Normativas do Anel Circular (`[P]`)

1. **Unidade de indexação:** Todos os ponteiros (`in_wpos`, `in_rpos`, `out_wpos`, `out_rpos`) são índices em **words de 32 bits** no intervalo `[0, 1019]` (`modulo 1020`).
2. **Alinhamento a 32 bits:** Uma mensagem de `size` bytes consome exatamente:
   ```text
   words_consumidas = 1 + ceil(size / 4) = (4 + size + 3) / 4
   ```
   Se `size` não for múltiplo de 4, os últimos `1..3` bytes ocupam o início da última word e o índice avança 1 word inteira (arredondamento para cima).
3. **Wrap-around por word:** Cada word (do cabeçalho e do payload) é lida/escrita em `data[pos]` com `pos = (pos + 1) % 1020`. Uma mensagem pode cruzar a fronteira `1019 → 0` perfeitamente.
4. **Cálculo de espaço/dados disponíveis:**
   - **Palavras disponíveis para leitura (consumidor):**
     ```c
     int avail = (int)wpos - (int)rpos;
     if (avail < 0) avail += 1020;
     /* avail == 0 significa fila vazia */
     ```
   - **Espaço livre para escrita (produtor):**
     ```c
     int free_words = (int)rpos - (int)wpos;
     if (free_words <= 0) free_words += 1020;
     /* Precisa de free_words > words_consumidas (sempre manter >= 1 word livre para distinguir cheia de vazia!) */
     ```
5. **Ordem de atualização e barreiras (lado Firmware):**
   - **Ao enviar (FW → Host):**
     1. Gravar `header` e payload em `mve->out_data[]`.
     2. Atualizar `mve->out_wpos = novo_wpos`.
     3. Executar sincronização `CR14 = 0` (`mtcr r, 0x0E`).
     4. Disparar IRQ para o host escrevendo `1` em `0xFF02B028` (`[O]`).
   - **Ao consumir (Host → FW):**
     1. Ler `host->in_wpos`; se diferente de `mve->in_rpos`, ler `header` e `header.size` bytes de `host->in_data[]`.
     2. Atualizar `mve->in_rpos = novo_rpos` e executar `CR14 = 0` para que o host veja o espaço liberado.
6. **Validação de códigos pelo driver host (`[P]`):** O driver descarta leituras cujo `header.code` esteja fora de `[1001..1013]`, `[2001..2019]` ou `[3001..3004]`.

---

## 3. Canal RPC (`0x1007_F000`) (`[P][O]`)

O canal RPC permite ao firmware solicitar serviços síncronos ao driver host — principalmente alocação, redimensionamento e liberação de memória na MMU da sessão (pois o firmware não possui alocador físico próprio) e impressão de debug.

### 3.1 Estrutura `struct mve_rpc_communication_area` (1024 Bytes)

```c
#define MVE_RPC_AREA_SIZE_IN_WORDS 256
#define MVE_RPC_DATA_SIZE_IN_WORDS 253

struct mve_rpc_communication_area {
    volatile uint32_t state;   /* offset 0x00: 0=FREE, 1=PARAM, 2=RETURN */
    volatile uint32_t call_id; /* offset 0x04: 1=DEBUG_PRINTF, 2=MEM_ALLOC, 3=MEM_RESIZE, 4=MEM_FREE */
    volatile uint32_t size;    /* offset 0x08: tamanho em bytes dos dados em params */
    union mve_rpc_params {     /* offset 0x0C..0x3FF (1012 bytes = 253 words) */
        volatile uint32_t data[253];
        struct {
            char string[1012]; /* string ASCII terminada em '\0' */
        } debug_print;
        struct {
            uint32_t size;           /* +0x00: tamanho inicial desejado em bytes */
            uint32_t max_size;       /* +0x04: capacidade máxima para futuros MEM_RESIZE */
            uint8_t  region;         /* +0x08: 0=MVE_MEM_REGION_PROTECTED, 1=MVE_MEM_REGION_OUTBUF/FRAMEBUF */
            uint8_t  log2_alignment; /* +0x09: alinhamento mínimo = 2^(log2_alignment) bytes */
        } mem_alloc;                 /* size esperado = 12 bytes */
        struct {
            uint32_t ve_pointer;     /* +0x00: VA anteriormente retornado por MEM_ALLOC */
            uint32_t new_size;       /* +0x04: novo tamanho em bytes */
        } mem_resize;                /* size esperado = 8 bytes */
        struct {
            uint32_t ve_pointer;     /* +0x00: VA a liberar */
        } mem_free;                  /* size esperado = 4 bytes */
    } params;
};
```

### 3.2 Protocolo de Handshake RPC (Máquina de Estados)

```text
  [state = FREE (0)]
         │
         │ Firmware preenche call_id, size, params
         │ Firmware escreve state = PARAM (1) + CR14 = 0 + IRQ (0xFF02B028 = 1)
         ▼
  [state = PARAM (1)]
         │
         │ Host atende a chamada no irq_bottom:
         │   - DEBUG_PRINTF (1): imprime params.debug_print.string no log do kernel
         │   - MEM_ALLOC    (2): aloca páginas, mapeia na MMU (região + alinhamento),
         │                       grava size = 4, params.data[0] = VA (ou 0 em falha)
         │   - MEM_RESIZE   (3): redimensiona páginas, faz FLUSH_ALL na MMU,
         │                       grava size = 4, params.data[0] = VA (ou 0 em falha)
         │   - MEM_FREE     (4): desmapeia e libera páginas, faz FLUSH_ALL na MMU,
         │                       grava size = 0
         │ Host executa wmb() + state = RETURN (2) + dma_sync + IRQHOST = 1
         ▼
  [state = RETURN (2)]
         │
         │ Firmware lê resultado (params.data[0] para ALLOC/RESIZE)
         │ Firmware escreve state = FREE (0) antes da próxima chamada RPC
         ▼
  [state = FREE (0)]
```

### 3.3 Exemplo Real Observado em Hardware (`[O]`)

Durante uma sessão H.264 128×128 no K1:
- **Pedido do Firmware:** `state = 1 (PARAM)`, `call_id = 2 (MEM_ALLOC)`, `size = 12`, `mem_alloc = { size = 0x20000 (128 KiB), max_size = 0x4000000 (64 MiB), region = 0 (PROTECTED), log2_alignment = 22 (4 MiB) }`.
- **Resposta do Host:** `state = 2 (RETURN)`, `call_id = 2`, `size = 4`, `params.data[0] = 0x20400000` (VA alinhado a 4 MiB dentro de `PROTECTED`).

---

## 4. RAM de Log Circular do Firmware (`0x1010_0000`, 512 KiB) (`[P]`)

O driver mapeia 512 KiB contíguos em `MVE_FW_PRINT_RAM_ADDR` (`0x1010_0000`) para mensagens de diagnóstico assíncronas do firmware (lidas pelo driver quando `MVX_FW_DEBUG_ENABLE` está ativo):

### 4.1 Cabeçalho (`struct mve_fw_ram_print_head_aera`, 128 Bytes em `0x1010_0000`)

| Offset | Tipo | Campo | Escritor | Função |
|---:|---|---|---|---|
| `0x00` | `volatile uint32_t` | `rd_cnt` | Host | Contador monotônico de mensagens já lidas pelo host |
| `0x04` | `uint32_t[15]` | `reserved0` | — | Padding para separar linha de cache (64 B) |
| `0x40` | `volatile uint32_t` | `flag` | Firmware | Número mágico de validação: **`0x11223356`** (`RAM_PRINT_FLAG`) |
| `0x44` | `volatile uint32_t` | `index` | Firmware | Índice atual de escrita (`0 .. RAM_PRINT_BUF_CNT - 1`) |
| `0x48` | `volatile uint32_t` | `wr_cnt` | Firmware | Contador monotônico de mensagens escritas pelo firmware |
| `0x4C` | `uint32_t[13]` | `reserved1` | — | Padding até 128 bytes (`0x80`) |

### 4.2 Slots de Mensagens (`0x1010_0080` – `0x1017_FFFF`)

- **Tamanho de cada slot:** `RAM_PRINT_MAX_LEN = 128` bytes (string ASCII terminada em `\0`).
- **Número total de slots:** `RAM_PRINT_BUF_CNT = (524288 / 128) - 1 = 4095` slots (`slot 0` em `0x1010_0080`, `slot k` em `0x1010_0080 + k * 128`).
- **Regra de escrita do firmware:**
  1. Inicializar `flag = 0x11223356`, `index = 0`, `wr_cnt = 0` no boot.
  2. Para registrar uma linha: copiar texto (máx. 127 chars + `\0`) no slot `idx = wr_cnt % 4095` (endereço `0x1010_0080 + idx * 128`), atualizar `index = idx` e incrementar `wr_cnt` (com barreira antes de incrementar `wr_cnt`).
  3. O driver lê todas as entradas de `rd_cnt` até `wr_cnt` sempre que `flag == 0x11223356` e `index < 4095`, atualizando `rd_cnt` após drenar.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
