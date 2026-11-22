<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# MMU da VPU e Modelo de Memória Virtual (MVE 3.x)

Este documento especifica a arquitetura da MMU de 2 níveis da VPU Linlon V5, o layout exato das entradas da tabela de páginas (PTE), o mapa completo do espaço de endereçamento virtual de 32 bits definido pelo protocolo MVE 3.x e as regras de coerência entre CPU hospedeira e VPU.

---

## 1. Arquitetura da MMU de 2 Níveis (`[P]`)

A VPU traduz endereços virtuais de **32 bits** (`mvx_mmu_va`) em endereços físicos de até **40 bits** no barramento AXI (restritos a 34 bits pelo `DMA_BIT_MASK(34)` no K1), usando páginas de **4 KiB** (`MVE_PAGE_SIZE = 4096`, `MVE_PAGE_SHIFT = 12`).

- **Níveis de page walk:** `2` (`L1` e `L2`).
- **Entradas por página de tabela:** `1024` (`4096 / 4 bytes`, `MVE_INDEX_SHIFT = 10`).
- **Isolamento:** Cada LSID possui sua própria tabela `L1`, cujo PTE raiz é escrito em `LSID.MMU_CTRL` (`+0x04`).

### 1.1 Decomposição do Endereço Virtual (32 bits)

```text
 31                22 21                12 11                     0
+--------------------+--------------------+------------------------+
|   Índice L1 (10b)  |   Índice L2 (10b)  |   Page Offset (12b)    |
+--------------------+--------------------+------------------------+
      0..1023              0..1023               0..4095
```

- `L1_index = (va >> 22) & 0x3FF` (cada entrada L1 cobre `4 MiB`).
- `L2_index = (va >> 12) & 0x3FF` (cada entrada L2 cobre `4 KiB`).
- `page_offset = va & 0xFFF`.

### 1.2 Formato do PTE de 32 bits (`mvx_mmu_pte`)

O mesmo formato de 32 bits é usado em `MMU_CTRL`, nas entradas da tabela `L1` (apontando para páginas `L2`) e nas entradas da tabela `L2` (apontando para páginas de código/dados):

```text
 31 30 29                                                2 1  0
+-----+---------------------------------------------------+----+
| AT  |                  PA [39:12] (28 bits)             | AP |
+-----+---------------------------------------------------+----+
```

| Campo | Bits | Shift / Máscara | Significado |
|---|---|---|---|
| `AT` | `[31:30]` | `shift = 30`, `mask = 0x3` | Atributo de memória / seleciona qual registrador `BUSATTR_<AT>` (`0..3`) define os atributos AXI da transação |
| `PA` | `[29:2]` | `shift = 2`, `mask = 0x0FFF_FFFF` | Número da página física no espaço do barramento VPU (`pa_vpu >> 12`, cobrindo `PA[39:12]`) |
| `AP` | `[1:0]` | `shift = 0`, `mask = 0x3` | Permissão de acesso (`0` = inválida/sem acesso → gera `MMU ABORT`) |

**Construção normativa de um PTE:**
```c
pte = ((attr & 0x3u) << 30) | (((pa_vpu >> 12) & 0x0FFFFFFFu) << 2) | (access & 0x3u);
```

### 1.3 Valores de `AT` (`enum mvx_mmu_attr`) e `AP` (`enum mvx_mmu_access`)

| `AT` | Nome | Uso Típico | `BUSATTR` Associado |
|---:|---|---|---|
| `0` | `MVX_ATTR_PRIVATE` | Tabelas L1/L2, texto executável do firmware, BSS privado e `master_rw` | `BUSATTR_0` |
| `1` | `MVX_ATTR_REFFRAME` | Quadros de referência / buffers AFBC | `BUSATTR_1` |
| `2` | `MVX_ATTR_SHARED_RO` | Memória compartilhada somente leitura | `BUSATTR_2` |
| `3` | `MVX_ATTR_SHARED_RW` | Filas de comunicação MVE, área RPC, `ram_print` e buffers compartilhados | `BUSATTR_3` |

| `AP` | Nome | Permissão |
|---:|---|---|
| `0` | `MVX_ACCESS_NO` | Sem acesso (entrada inválida; gera `MMU ABORT`) |
| `1` | `MVX_ACCESS_READ_ONLY` | Somente leitura (também usado em entradas L1 que apontam para tabelas L2) |
| `2` | `MVX_ACCESS_EXECUTABLE` | Executável / leitura de instruções (usado no segmento `text` do firmware) |
| `3` | `MVX_ACCESS_READ_WRITE` | Leitura e escrita (usado em BSS, filas, RPC e framebuffers) |

### 1.4 Invalidação de TLB

- Sempre que o host mapeia/redimensiona/libera páginas em uma sessão ativa (ex.: `RPC MEM_RESIZE`, `RPC MEM_FREE` ou mapeamento de novos buffers), deve escrever `0` em `LSID.FLUSH_ALL` (`+0x10`).
- Ao atualizar qualquer entrada L1 ou L2, o host deve sincronizar a cache da CPU para o dispositivo (`dma_sync_single_for_device` com `phys_cpu2vpu`).

---

## 2. Mapa de Memória Virtual MVE 3.x (`[P][O]`)

O espaço virtual de 32 bits (`0x0000_0000` – `0xFFFF_FFFF`) é particionado de forma fixa pelo contrato `mve_protocol_def.h`:

| Região | Início (VA) | Fim (não inclusivo) | Tamanho | Atributo / Acesso | Função |
|---|---|---|---:|---|---|
| **FW Instância 0** | `0x0000_0000` | `0x0010_0000` | 1 MiB | `PRIVATE` / `X` (`text`), `RW` (`bss`/`master_rw`) | Core físico 0 (código em `+0x1000`) |
| **FW Instância 1** | `0x0100_0000` | `0x0110_0000` | 1 MiB | `PRIVATE` / `X`, `RW` | Core físico 1 (código em `0x0100_1000`) |
| **FW Instância 2** | `0x0200_0000` | `0x0210_0000` | 1 MiB | `PRIVATE` / `X`, `RW` | Core físico 2 (K3; código em `0x0200_1000`) |
| **FW Instância 3** | `0x0300_0000` | `0x0310_0000` | 1 MiB | `PRIVATE` / `X`, `RW` | Core físico 3 (K3; código em `0x0300_1000`) |
| **FW Instâncias 4–7** | `0x0400_0000` – `0x0700_0000` | `+0x0010_0000` cada | 1 MiB × 4 | `PRIVATE` / `X`, `RW` | Escalabilidade até 8 cores |
| **MSG_INQ (Host → FW)** | `0x1007_9000` | `0x1007_A000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_host` (requisições) |
| **MSG_OUTQ (FW → Host)** | `0x1007_A000` | `0x1007_B000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_mve` (respostas) |
| **BUF_INQ (Host → FW)** | `0x1007_B000` | `0x1007_C000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_host` (buffers de entrada) |
| **BUF_INRQ (FW → Host)** | `0x1007_C000` | `0x1007_D000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_mve` (buffers de entrada devolvidos) |
| **BUF_OUTQ (Host → FW)** | `0x1007_D000` | `0x1007_E000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_host` (buffers de saída) |
| **BUF_OUTRQ (FW → Host)** | `0x1007_E000` | `0x1007_F000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_comm_area_mve` (buffers de saída devolvidos) |
| **RPC** | `0x1007_F000` | `0x1008_0000` | 4 KiB | `SHARED_RW` / `RW` | `struct mve_rpc_communication_area` |
| **FW Print RAM** | `0x1010_0000` | `0x1018_0000` | 512 KiB | `SHARED_RW` / `RW` | Buffer circular de logs de texto do firmware |
| **PROTECTED** | `0x2000_0000` | `0x7000_0000` | 1,25 GiB | `PRIVATE` ou `SHARED_RW` / `RW` | Bitstreams, buffers intermediários (`IntBuf`), alocações RPC `region=0` |
| **FRAMEBUF** | `0x7000_0000` | `0xF000_0000` | 2,0 GiB | `REFFRAME` ou `SHARED_RW` / `RW` | Framebuffers planar/AFBC, bancos de referência, alocações RPC `region=1` |
| **Uso Interno MVE** | `0xF000_0000` | `0x1_0000_0000` | 256 MiB | Interno (não traduzido via DRAM) | MMIO interno das CPUs RASC e SRAMs internas (`0xFF00_0000`–`0xFF2F_8FFF`) |

> **Atenção à página zero de cada instância (`[P]`):** O endereço `core_base + 0x0000` (`0x0000_0000`, `0x0100_0000`, ...) **não é mapeado** (`AP = 0`), atuando como página-guarda contra desreferência de ponteiro `NULL`. O código executável do firmware começa em **`core_base + 0x1000`** (`FW_TEXT_BASE_ADDR = 0x1000`).

---

## 3. Regras de Regiões de Memória Exigidas pelo Contrato (`[P][D]`)

O firmware deve validar os endereços virtuais recebidos do host e retornados pelo RPC contra as faixas acima:

1. **Alocações RPC (`mve_rpc_params.mem_alloc`):**
   - `region = 0` (`MVE_MEM_REGION_PROTECTED`): o VA retornado pelo host **deve** cair em `[0x2000_0000, 0x7000_0000)` e ser múltiplo de `2^(log2_alignment)`.
   - `region = 1` (`MVE_MEM_REGION_OUTBUF` / `FRAMEBUF`): o VA retornado pelo host **deve** cair em `[0x7000_0000, 0xF000_0000)` e ser múltiplo de `2^(log2_alignment)`.
2. **Buffers de Frame (`mve_buffer_frame`):**
   - Planos válidos (`plane_top[]`, `plane_bot[]`, `afbc.plane[]`) devem residir na região `FRAMEBUF` (`[0x7000_0000, 0xF000_0000)`) — ou em `PROTECTED` em sessões seguras. Planos não utilizados devem ter endereço `0`.
3. **Buffers de Bitstream (`mve_buffer_bitstream`):**
   - `bitstream_buf_addr` reside tipicamente na região `PROTECTED` (`[0x2000_0000, 0x7000_0000)`, observado em `0x2000_0000` no K1 `[O]`).
4. **Páginas de BSS não setadas no bitmap:**
   - Qualquer página dentro do intervalo do bitmap de BSS cujo bit seja `0` (e fora de `master_rw`) permanece **não mapeada** na MMU; qualquer acesso gera `MMU ABORT`.

---

## 4. Visão Multi-Core das Instâncias de Firmware (`[P][O]`)

Quando uma sessão usa múltiplos cores (ex.: `ncores = 2` no K1 ou `4` no K3):

- **Segmento `text`:** As **mesmas páginas físicas** são mapeadas simultaneamente em `0x0000_1000`, `0x0100_1000`, ..., `(ncores-1)*0x0100_0000 + 0x1000` com permissão `MVX_ACCESS_EXECUTABLE`.
- **Segmento `bss` privado:** Cada core `c` recebe seu próprio conjunto exclusivo de páginas físicas zeradas em `c * 0x0100_0000 + va`.
- **Segmento `master_rw` compartilhado:** As **mesmas páginas físicas** são mapeadas em `c * 0x0100_0000 + va` para todos os cores `c = 0 .. ncores-1` com permissão `MVX_ACCESS_READ_WRITE`.
- **Filas MVE e RPC (`0x1007_9000`–`0x1007_F000`):** Endereços absolutos únicos compartilhados entre o host e todos os cores da sessão.

**Consequência obrigatória para o firmware (`[O]`):** Como o core `c` executa em `c * 0x0100_0000 + 0x1000`, todo acesso a variáveis globais no BSS privado, pilhas e tabelas locais **deve** usar endereçamento **PC-relativo (`lpc`)**, e nunca constantes absolutas fixas para a instância `0`. Endereços fixos absolutos só devem ser usados para as regiões globais (`0x1007_9000`–`0x1018_0000`, `PROTECTED`, `FRAMEBUF` e MMIO interno `0xFF00_0000+`).

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
