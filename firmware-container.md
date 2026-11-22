<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Formato do Contêiner de Firmware (`.fwb`) e Contrato de Carga

Este documento especifica o formato binário `.fwb` esperado pelo driver Linux `mvx`, as validações executadas durante o carregamento, o mapeamento na MMU e as regras obrigatórias para que um firmware compilado em sala limpa boote corretamente no hardware real do SpacemiT K1/K3.

---

## 1. Resolução do Caminho do Arquivo `.fwb` (`[P]`)

O driver monta o caminho relativo em `/lib/firmware/` (ou `/usr/lib/firmware/`) a partir dos valores lidos em `HARDWARE_ID`:

```text
linlon-<hw_id_str>-<revision>-<patch>/<codec><dir>.fwb
```

Para o SpacemiT K1 e o SpacemiT K3 (`HARDWARE_ID = 0x56635002`, `MVE_v52_v76`, `revision = 80`, `patch = 2`):

```text
linlon-v52_v76-80-2/<codec><dec|enc>.fwb
```

Exemplos: `linlon-v52_v76-80-2/h264dec.fwb`, `linlon-v52_v76-80-2/jpegdec.fwb`, `linlon-v52_v76-80-2/hevcenc.fwb`. A lista completa de mapeamentos de codec (incluindo H.263 → `mpeg4dec.fwb`) está em [codecs-and-capabilities.md](codecs-and-capabilities.md).

---

## 2. Estrutura do Cabeçalho (`struct mvx_fw_header`, 180 Bytes) (`[P]`)

O arquivo `.fwb` é uma **imagem plana (*flat binary*)** sem cabeçalho ELF, sem relocações e sem compressão. Seus primeiros **180 bytes (`0x00`–`0xB3`)** formam `struct mvx_fw_header` (todos os inteiros em **little-endian**):

| Offset | Tamanho | Tipo | Campo | Contrato / Descrição |
|---:|---:|---|---|---|
| `0x00` | 4 | `uint32_t` | `rasc_jmp` | **Vetor de reset executável.** São os 4 primeiros bytes executados pela CPU RASC em `VA 0x1000` (deve conter um salto `br` para além do header; ver §4). |
| `0x04` | 1 | `uint8_t` | `protocol_minor` | Versão minor do protocolo MVE. Deve ser **`5`** para MVE 3.5 (com `major=3`, `minor>=1` habilita `IDLE_ACK`). |
| `0x05` | 1 | `uint8_t` | `protocol_major` | Versão major do protocolo MVE. Deve ser **`3`** (driver aceita `2` ou `3`; `3` ativa mapa MVE 3.x e H.264 High 10). |
| `0x06` | 2 | `uint8_t[2]` | `reserved` | Reservado (`0` no contrato público). |
| `0x08` | 56 | `uint8_t[56]` | `info_string` | String ASCII legível terminada em `\0` descrevendo alvo e codec (exibida no log do driver). |
| `0x40` | 8 | `uint8_t[8]` | `part_number` | Identificador ASCII de até 8 bytes (ex.: `"H264DEC"`). |
| `0x48` | 8 | `uint8_t[8]` | `svn_revision` | Identificador ASCII de revisão de build (8 bytes). |
| `0x50` | 16 | `uint8_t[16]` | `version_string` | String ASCII de versão do firmware (até 16 bytes). |
| `0x60` | 4 | `uint32_t` | `text_length` | **Comprimento total em bytes a copiar a partir do offset `0x00` do arquivo** (ou seja, **inclui os 180 bytes do header + padding + todo o código/rodata**). |
| `0x64` | 4 | `uint32_t` | `bss_start_address` | Endereço virtual inicial do segmento BSS (relativo a `core_base`, **alinhado a 4 KiB** e estritamente após o fim do segmento `text`). |
| `0x68` | 4 | `uint32_t` | `bss_bitmap_size` | Número de páginas de 4 KiB descritas por `bss_bitmap` (máximo `16 * 32 = 512` páginas = 2 MiB; dentro da janela de 1 MiB da instância o limite prático é menor). |
| `0x6C` | 64 | `uint32_t[16]` | `bss_bitmap` | Bitmap little-endian (bit `i` = `(bss_bitmap[i >> 5] >> (i & 31)) & 1`). Se `1` (e fora de `master_rw`), aloca uma página de BSS **privada por core** em `core_base + bss_start_address + i * 0x1000`. |
| `0xAC` | 4 | `uint32_t` | `master_rw_start_address` | Endereço virtual inicial (relativo a `core_base`, alinhado a 4 KiB) da região de BSS **compartilhada** entre todos os cores da sessão. |
| `0xB0` | 4 | `uint32_t` | `master_rw_size` | Tamanho em bytes da região compartilhada `master_rw` (múltiplo de 4 KiB; `0` se não usada). |

---

## 3. Algoritmo Exato de Validação e Carga pelo Driver (`[P]`)

Ao carregar um `.fwb` em modo não-seguro (`mvx_firmware_cache.c` e `mvx_firmware.c`):

1. **Validação:**
   - Rejeita (`-EFAULT`) se `fw->size < 180` (`sizeof(struct mvx_fw_header)`).
   - Rejeita (`-EFAULT`) se `header->text_length > fw->size`.
   - Rejeita (`-EINVAL`) se `protocol_major` não for `2` ou `3`.
2. **Contagem de páginas:**
   - `text_cnt = ceil(header->text_length / 4096)`.
   - Percorre `i = 0 .. header->bss_bitmap_size - 1` com `va = header->bss_start_address + i * 4096`:
     - Se `va >= master_rw_start_address && va < master_rw_start_address + master_rw_size`: incrementa `sbss_cnt` (note que **esta verificação tem precedência sobre o bit do `bss_bitmap`**).
     - Senão, se `bit(i, bss_bitmap) == 1`: incrementa `bss_cnt`.
     - Senão: página permanece não mapeada (página-guarda).
3. **Alocação de memória física (todas as páginas zeradas):**
   - `fw->text`: `text_cnt` páginas (únicas, compartilhadas RO/X por todos os cores).
   - `fw->bss`: `bss_cnt * ncores` páginas (privadas por core).
   - `fw->bss_shared`: `sbss_cnt` páginas (únicas, compartilhadas RW por todos os cores).
4. **Mapeamento MMU por core (`core = 0 .. ncores - 1`, `fw_base = core * 0x0100_0000`):**
   - Mapeia `fw->text` em `fw_base + 0x1000` com `MVX_ATTR_PRIVATE`, `MVX_ACCESS_EXECUTABLE`.
   - Mapeia cada página de `master_rw` em `fw_base + va` apontando para `fw->bss_shared` com `MVX_ATTR_PRIVATE`, `MVX_ACCESS_READ_WRITE`.
   - Mapeia cada página privada com bit `1` em `fw_base + va` apontando para `fw->bss[core * bss_cnt + k]` com `MVX_ATTR_PRIVATE`, `MVX_ACCESS_READ_WRITE`.
5. **Cópia da imagem:**
   - Copia `header->text_length` bytes a partir do **offset `0x00` do arquivo `.fwb`** para `VA 0x1000` (como as páginas físicas de `fw->text` são compartilhadas, a cópia fica imediatamente visível em `0x0000_1000`, `0x0100_1000`, ..., `(ncores-1)*0x0100_0000 + 0x1000`).
   - Executa `dma_sync_single_for_device` para garantir coerência com a DRAM.

---

## 4. Regras Obrigatórias para Construção de um `.fwb` Clean-Room (`[P][O]`)

A execução experimental em hardware real do K1 comprovou quatro regras indispensáveis; violar qualquer uma impede o firmware de executar ou causa falha silenciosa:

### Regra 1 — `text_length` DEVE incluir os 180 bytes do header (`[P][O]`)
Como o driver copia `text_length` bytes a partir do byte `0` do arquivo, definir `text_length` apenas como o tamanho do código após o header truncará os últimos 180 bytes da imagem em memória. Defina sempre `text_length = tamanho_total_do_arquivo` (ou `offset_do_codigo + tamanho_do_codigo_e_rodata`).

### Regra 2 — O offset `0x00` do arquivo DEVE conter um trampolim `br` (`[O]`)
O hardware inicia a execução da CPU RASC0 no primeiro byte do segmento `text` (`VA 0x1000` = offset `0x00` do arquivo), onde reside o próprio `struct mvx_fw_header`. Portanto, os 4 bytes de `rasc_jmp` no offset `0x00` devem ser uma instrução válida de salto incondicional RASC (`br`) desviando para o ponto de entrada após o header (`offset >= 0xB4`, por exemplo `0xB4` ou `0xC0`, com alinhamento adequado):
- Exemplo de codificação RASC de `br` com sufixo de 2 bytes (3 bytes de instrução + 1 byte de padding `0x00`):
  - Byte `0`: `0x5E` (`br`, 2-bit range field = `1` → sufixo signed de 2 bytes).
  - Bytes `1..2`: `delta = (target_offset - 1)` em little-endian de 16 bits (`target = pc + 1 + delta`, já que `base_len = 1` para `br`).
  - Byte `3`: `0x00` (padding).
  - Nota: para saltar do offset `0x00` ao offset `0xB4`, `delta = 0xB4 - 1 = 0x00B3` → bytes `5E B3 00 00`. Para saltar ao offset `0xC0`, `delta = 0xC0 - 1 = 0x00BF` → bytes `5E BF 00 00`.

### Regra 3 — Alinhamento entre Linker Script e Offset do Payload (`[O]`)
Todo endereço de código e dados no linker script deve corresponder à relação fixa:
```text
VA_na_instancia_0 = 0x1000 + offset_no_arquivo
```
Se o linker gera endereços base `0x0000` (offset de arquivo) e utiliza exclusivamente instruções PC-relativas (`br`, `bsr`, `lpc`), a imagem funciona tanto em `0x0000_1000` quanto em `0x0100_1000`, `0x0200_1000`, `0x0300_1000` sem necessidade de relocação. Caso existam tabelas de ponteiros absolutos em `.rodata`, considere o deslocamento de `0x1000` (e cuidado com instâncias `core >= 1`, cujo base difere em `core * 0x0100_0000`).

### Regra 4 — Inicialização PC-Relativa da Pilha (`sp`) (`[O]`)
A pilha de cada CPU RASC reside no BSS privado da instância. Carregar `sp` com uma constante absoluta fixa (ex.: `mov sp, #0x80000`) falha em qualquer instância `core >= 1` (onde o BSS privado é deslocado em `core * 0x0100_0000`). Sempre inicialize `sp` de forma relativa ao PC:
```asm
lpc sp, stack_top
```

---

## 5. Layout Recomendado de uma Imagem `.fwb` Clean-Room (`[D]`)

```text
Offset no arquivo    VA (Core c, base = c * 0x0100_0000)    Conteúdo
─────────────────    ───────────────────────────────────    ──────────────────────────────
(não mapeado)        base + 0x0000 .. base + 0x0FFF         Página-guarda (AP = 0)
0x0000 .. 0x0003     base + 0x1000 .. base + 0x1003         Trampolim `br _start` (campo rasc_jmp)
0x0004 .. 0x00B3     base + 0x1004 .. base + 0x10B3         Restante de struct mvx_fw_header
0x00B4 .. (alinh.)   base + 0x10B4 ..                       Padding opcional até _start
_start .. text_end   base + 0x1000 + _start ..              .text + .rodata (executável/RO)
(fora do arquivo)    base + bss_start_address ..            .bss privado por core + pilhas (RW, zerado)
(fora do arquivo)    base + master_rw_start_address ..      master_rw compartilhado entre cores (RW, zerado)
```

- Lembre-se de garantir que `bss_start_address` (alinhado a `0x1000`) seja `>= 0x1000 + align_up(text_length, 0x1000)` para não colidir com o mapeamento do segmento `text`!
- O driver zera todas as páginas de `bss` e `bss_shared` ao alocá-las; se o firmware precisar de variáveis globais inicializadas com valores não-zero (`.data`), o startup deve copiá-las de `.rodata` (no segmento `text`) para `.bss` antes de entrar em `main()`.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
