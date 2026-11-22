<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Protocolo MVE 3.5 — Descritores de Buffer e Formatos de Pixel

Este documento especifica os quatro códigos de buffer (`3001`–`3004`) traficados nas filas `BUF_INQ`/`BUF_INRQ` e `BUF_OUTQ`/`BUF_OUTRQ`, os layouts exatos das estruturas de frame (Planar e AFBC), bitstream, parâmetros inline e buffers gerais, o bitfield de 16 bits `format` e as validações exigidas pelo contrato (`mve_protocol_def.h`, `[P][D]`).

---

## 1. Códigos de Buffer (`mve_msg_header.code`) (`[P]`)

| Código | Macro Pública | Estrutura | Direção / Uso |
|---:|---|---|---|
| `3001` | `MVE_BUFFER_CODE_FRAME` | `struct mve_buffer_frame` | Frame de saída (decode) ou entrada (encode) |
| `3002` | `MVE_BUFFER_CODE_BITSTREAM` | `struct mve_buffer_bitstream` | Bitstream de entrada (decode) ou saída (encode) |
| `3003` | `MVE_BUFFER_CODE_PARAM` | `struct mve_buffer_param` | Parâmetro inline na fila de buffers (aplica-se aos buffers subsequentes) |
| `3004` | `MVE_BUFFER_CODE_GENERAL` | `struct mve_buffer_general` | Dados auxiliares estruturados (block configs de encode / AD stats de decode) |

> **Preservação de `host_handle` e `user_data_tag` (`[P]`):**
> - `host_handle` (64 bits) identifica o objeto de buffer no driver host e **nunca** deve ser modificado pelo firmware ao devolver o buffer.
> - `user_data_tag` (64 bits) associa entrada e saída: o firmware **deve** propagar o `user_data_tag` do buffer de bitstream para o(s) buffer(s) de frame produzidos (decode) e vice-versa (encode).

---

## 2. `struct mve_buffer_frame` (`3001`) (`[P]`)

```c
struct mve_buffer_frame_planar {
    uint32_t plane_top[3];       /* VAs dos planos Y, Cb, Cr (campo superior ou frame progressivo) */
    uint32_t plane_bot[3];       /* VAs dos planos Y, Cb, Cr (campo inferior, apenas interlace) */
    int32_t  stride[3];          /* Stride entre linhas em bytes (por plano) */
    uint16_t max_frame_width;    /* Largura máxima alocada no buffer */
    uint16_t max_frame_height;   /* Altura máxima alocada no buffer */
};                               /* sizeof = 40 bytes */

struct mve_buffer_frame_afbc {
    uint32_t plane[2];                     /* VAs: [0]=top/normal, [1]=bottom (interlace) ou downscaled */
    uint32_t alloc_bytes[2];               /* Tamanho alocado em bytes para cada plano */
    uint16_t cropx;                        /* Crop X em amostras de luma */
    uint16_t cropy;                        /* Crop Y em amostras de luma */
    uint16_t afbc_width_in_superblocks[2]; /* Largura em superblocos (16 ou 32 px) */
    uint32_t afbc_params;                  /* Bitmask MVE_BUFFER_FRAME_AFBC_* */
};                                         /* sizeof = 28 bytes */

struct mve_buffer_frame {
    uint64_t host_handle;          /* +0x00 */
    uint64_t user_data_tag;        /* +0x08 */
    uint32_t frame_flags;          /* +0x10 */
    uint16_t visible_frame_height; /* +0x14 */
    uint16_t visible_frame_width;  /* +0x16 */
    uint16_t format;               /* +0x18 (bitfield de 16 bits) */
    uint16_t reserved0;            /* +0x1A */
    union {                        /* +0x1C (40 bytes) */
        struct mve_buffer_frame_planar planar;
        struct mve_buffer_frame_afbc   afbc;
    } data;
    uint32_t reserved1;            /* +0x44 (força sizeof = 72 bytes, múltiplo de 8) */
};
```

### 2.1 Flags `frame_flags` (`MVE_BUFFER_FRAME_FLAG_*`)

| Máscara | Nome | Quem Seta (Dec / Enc) | Semântica |
|---|---|---|---|
| `0x0000_0001` | `INTERLACE` | Host / — | Buffer entrelaçado (campos top e bottom alocados) |
| `0x0000_0002` | `BOT_FIRST` | FW / — | Campo inferior deve ser exibido primeiro |
| `0x0000_0004` | `TOP_PRESENT` | FW / Host | Campo superior (ou frame progressivo completo) presente |
| `0x0000_0008` | `BOT_PRESENT` | FW / — | Campo inferior presente |
| `0x0000_0010` | `ROTATION_90` | Host / Host | Rotação 90° |
| `0x0000_0020` | `ROTATION_180` | Host / Host | Rotação 180° |
| `0x0000_0030` | `ROTATION_270` | Host / Host | Rotação 270° (não suportada para AFBC em decode) |
| `0x0000_00C0` | `SCALING_MASK` | Host / — | Fator de downscale fixo |
| `0x0000_0100` | `MIRROR_HORI` | — / Host | Espelhamento horizontal (apenas encode; proibido em decode) |
| `0x0000_0200` | `MIRROR_VERT` | — / Host | Espelhamento vertical (apenas encode; proibido em decode) |
| `0x0000_1000` | `REJECTED` | FW / — | Buffer pequeno demais; host deve realocar |
| `0x0000_2000` | `CORRUPT` | FW / — | Frame contém corrupção visual |
| `0x0000_4000` | `DECODE_ONLY` | FW / — | Frame decodificado mas não deve ser exibido |
| `0x0000_8000` | `REF_FRAME` | FW / — | Frame em uso como referência (host não deve alterar; re-enfileirar ou usar `RELEASE_REF_FRAME`) |
| `0x0001_0000` | `EOS` | FW / Host | Último frame do stream (*End of Stream*) |
| `0x00FE_0000` | `SCALING_MASKY` | Host / — | Escala vertical arbitrária (7 bits) |
| `0xFF00_0000` | `SCALING_MASKX` | Host / — | Escala horizontal arbitrária (8 bits) |

> **Nota do driver host (`[P]`):** No retorno de decode, se `visible_frame_width == 0`, `visible_frame_height == 0` ou `(frame_flags & (TOP_PRESENT | BOT_PRESENT)) == 0`, o driver zera `bytesused` dos planos do buffer V4L2. Portanto, todo frame decodificado válido **deve** setar `TOP_PRESENT` (e `BOT_PRESENT` se entrelaçado) além de `visible_frame_width/height`.

### 2.2 Parâmetros AFBC (`afbc_params`)

| Máscara | Nome (`MVE_BUFFER_FRAME_AFBC_*`) | Significado |
|---|---|---|
| `0x0000_0001` | `TILED_BODY` | Blocos de corpo AFBC em layout tiled |
| `0x0000_0002` | `TILED_HEADER` | Cabeçalhos AFBC em layout tiled |
| `0x0000_0004` | `32X8_SUPERBLOCK` | Superbloco 32×8 (default = 16×16; suportado apenas como entrada de encode) |
| `0x0000_0008` | `DN_FORCE_8BIT` | Plano AFBC downscaled em 8-bit mesmo se full-scale for 10-bit |
| `0x0000_0010` | `DN_FORCE_420` | Plano AFBC downscaled em 4:2:0 mesmo se full-scale for 4:2:2 |
| `0x0000_0020` | `STRIDE_SET_BY_MVE` | Decode: firmware escolhe `afbc_width_in_superblocks` se este bit estiver ativo ou valor for 0 |
| `0x0000_0040` | `BLOCK_SPLIT` | Habilita modo `block_split` de superbloco |

### 2.3 Codificação do Bitfield `format` (16 bits)

```text
  15  14 13  12 11 10 9   8 7        4  3  2       0
+----+-----+----+----+-----+----------+---+---------+
|  A | res | V  |res |  N  |    B     |res|    C    |
+----+-----+----+----+-----+----------+---+---------+
```

- `C` (`[2:0]`): Subamostragem de croma (`0=MONO`, `1=420`, `2=422`, `3=440`, `4=ARGB`).
- `B` (`[7:4]`): `bitdepth_max - 8` (`0` = 8-bit, `2` = 10-bit, `8` = 16-bit/P010/Y210).
- `N` (`[9:8]`): Número de planos (`1`, `2` ou `3`; `0` para AFBC).
- `V` (`[13:12]`): Variante do formato (ex.: `0=NV12`, `1=NV21`; `0..3` para ordem de canais RGBA/BGRA/ARGB/ABGR).
- `A` (`[15]`): `1` = AFBC comprimido, `0` = não-AFBC (planar/semi-planar/packed).

| Formato (`MVE_FORMAT_*`) | Valor Hex | Planos | Descrição |
|---|---:|---:|---|
| `YUV420_AFBC_8` | `0x8001` | AFBC | YUV 4:2:0 AFBC 8-bit |
| `YUV420_AFBC_10` | `0x8021` | AFBC | YUV 4:2:0 AFBC 10-bit |
| `YUV422_AFBC_8` | `0x8002` | AFBC | YUV 4:2:2 AFBC 8-bit |
| `YUV422_AFBC_10` | `0x8022` | AFBC | YUV 4:2:2 AFBC 10-bit |
| `YUV420_I420` | `0x0301` | 3 | Y, U, V planar 8-bit |
| `YUV420_I420_10` | `0x0321` | 3 | Y, U, V planar 10-bit |
| `YUV420_NV12` | `0x0201` | 2 | Y + UV intercalado 8-bit |
| `YUV420_NV21` | `0x1201` | 2 | Y + VU intercalado 8-bit |
| `YUV420_P010` | `0x0281` | 2 | Y + UV 10-bit em words de 16 bits (MSB-aligned) |
| `YUV420_Y0L2` | `0x0121` | 1 | YUV 4:2:0 10-bit packed |
| `YUV420_AQB1` | `0x1121` | 1 | YUV 4:2:0 10-bit packed variante |
| `YUV422_YUY2` | `0x0102` | 1 | YUYV 4:2:2 8-bit packed |
| `YUV422_UYVY` | `0x1102` | 1 | UYVY 4:2:2 8-bit packed |
| `YUV422_Y210` | `0x0182` | 1 | YUV 4:2:2 10-bit em 16 bits |
| `RGBA_8888` | `0x0104` | 1 | Entrada encode apenas |
| `BGRA_8888` | `0x1104` | 1 | Entrada encode apenas |
| `ARGB_8888` | `0x2104` | 1 | Entrada encode apenas |
| `ABGR_8888` | `0x3104` | 1 | Entrada encode apenas |
| `MBINFO` | `0x0001` | — | Debug apenas |
| `UNUSED` | `0x0000` | — | Não utilizado |

---

## 3. `struct mve_buffer_bitstream` (`3002`) (`[P]`)

```c
struct mve_buffer_bitstream {
    uint64_t host_handle;           /* +0x00 */
    uint64_t user_data_tag;         /* +0x08 */
    uint32_t bitstream_flags;       /* +0x10 */
    uint32_t bitstream_alloc_bytes; /* +0x14: capacidade total alocada em bytes */
    uint32_t bitstream_offset;      /* +0x18: offset inicial em bytes a partir de buf_addr */
    uint32_t bitstream_filled_len;  /* +0x1C: quantidade de bytes válidos */
    uint32_t bitstream_buf_addr;    /* +0x20: VA na MMU da VPU */
    uint8_t  frame_type;            /* +0x24: 0=I, 1=P, 2=B, 3=b */
    uint8_t  reserved[3];           /* +0x25..0x27 (sizeof = 40 bytes) */
};
```

### Flags `bitstream_flags` (`MVE_BUFFER_BITSTREAM_FLAG_*`)

| Máscara | Nome | Semântica |
|---|---|---|
| `0x0000_0001` | `EOS` | Fim de stream |
| `0x0000_0010` | `ENDOFFRAME` | Buffer termina na fronteira de um frame completo |
| `0x0000_0020` | `SYNCFRAME` | Frame de acesso aleatório / IDR / keyframe |
| `0x0000_0080` | `CODECCONFIG` | Contém dados de configuração (SPS/PPS/VPS) |
| `0x0000_0400` | `ENDOFSUBFRAME` | Termina na fronteira de um subframe/slice |

> **Convenção observada ao devolver bitstream em decode (`[O]`):** Ao consumir integralmente `N` bytes de entrada, o firmware devolve o descritor em `BUF_INRQ` com `bitstream_filled_len = 0` e `bitstream_offset += N`.

---

## 4. `struct mve_buffer_param` (`3003`) (`[P]`)

Enviado inline na mesma fila de buffers; `uint32_t type` seguido da união `data`:

| `type` | Macro (`MVE_BUFFER_PARAM_TYPE_*`) | Direção | Payload |
|---:|---|---|---|
| `2` | `QP` | Enc | `int32_t qp` (H.264/HEVC `0..51`; VP8 `0..63`/`0..127`; VP9 `0..63`/`0..255`) |
| `3` | `REGIONS` | Enc | `n_regions` (até 16) × `{ mbx_left, mbx_right, mby_top, mby_bottom, int16_t qp_delta }` |
| `5` | `DISPLAY_SIZE` | Dec saída | `{ uint16_t display_width; uint16_t display_height; }` |
| `6` | `RANGE_MAP` | Dec saída | `{ luma_map_enabled, luma_map_value, chroma_map_enabled, chroma_map_value }` (VC-1 AP) |
| `9` | `FRAME_RATE` | Enc | `uint32_t arg` em Q16 fps (ex.: `0x001E0000` = 30,0 fps) |
| `10` | `RATE_CONTROL` | Enc | `{ rate_control_mode (0=OFF,1=STD,2=VAR,3=CONST,4=CVAR), target_bitrate, maximum_bitrate }` |
| `12`/`13`/`14` | `QP_I` / `QP_P` / `QP_B` | Enc | `int32_t qp` por tipo de quadro (com rate control desligado) |
| `15` | `COLOUR_DESCRIPTION` | Dec/Enc | Metadados HDR/VUI (`flags` bit 0=mastering display, bit 1=CLL; range 0/1/2; primaries/transfer/matrix; mastering primaries/luminance; max/avg CLL; SAR; timing) |
| `16` | `FRAME_PACKING` | Enc | Metadados estereoscópicos SEI (HEVC D.3.16) |
| `17` | `FRAME_FIELD_INFO` | Dec saída | `{ pic_struct, source_scan_type, duplicate_flag }` |
| `18` | `GOP_RESET` | Enc | Sem payload extra; força reinício de GOP |
| `19` | `DPB_HELD_FRAMES` | Dec saída | `uint32_t arg`: contagem de frames prontos retidos no DPB para reordenação |
| `20` | `CHANGE_RECTANGLES` | Enc | `n_rectangles` (0..2) × `{ x_left, x_right, y_top, y_bottom }` (aplica-se apenas ao próximo frame) |
| `21` | `RATE_CONTROL_QP_RANGE` | Enc | `{ int32_t qp_min; int32_t qp_max; }` |
| `23` | `RATE_CONTROL_HRD_BUF_SIZE` | Enc | `uint32_t arg` (tamanho do buffer HRD) |
| `25` | `RATE_CONTROL_QP_RANGE_I` | Enc | `{ int32_t qp_min; int32_t qp_max; }` específico para I-frames |
| `26` | `SEI_USER_DATA_UNREGISTERED` | Dec/Enc | `{ flags (bit 0=valid), uuid[16], user_data[221], user_data_len }` |

---

## 5. `struct mve_buffer_general` (`3004`) (`[P]`)

Cabeçalho `struct mve_buffer_general_hdr` (32 bytes): `{ uint64_t host_handle, uint64_t user_data_tag, uint32_t buffer_ptr, uint32_t buffer_size, uint16_t type, uint16_t config_size, uint32_t reserved }`, seguido de `config` de tamanho `config_size`:

- `type = 1` (`MVE_BUFFER_GENERAL_TYPE_BLOCK_CONFIGS`, entrada de encode): `blk_cfg_type` (`0x00=NONE`, `0xFF=ROW_UNCOMP`), `{ n_cols_minus1, n_rows_minus1 }`, apontando para vetor de `mve_block_param_record` (`uint16_t qp_delta` com quatro deltas de 4 bits para um quad 32×32 + `uint8_t force`: `0=NONE`, `1=FORCE_QP`).
- `type = 2` (`MVE_BUFFER_GENERAL_TYPE_AD_STATS`, saída de decode): `{ uint32_t frame_averages (Y[11:0], Cb[21:12], Cr[31:22]), uint16_t thumbnail_width, uint16_t thumbnail_height, uint8_t ad_stats_flags (0=PROG, 1=INTERLACED) }`.

---

## 6. Regras de Validação de Buffers Exigidas do Firmware (`[P][D]`)

1. Rejeitar com `MVE_ERROR_INVALID_BUFFER` (`6`) se um descritor chegar no canal incorreto (ex.: `FRAME` em `BUF_INQ` de decode) ou com tipo `mve_buffer_general` inválido/não suportado.
2. Em buffers de frame de saída (decode):
   - Validar formato suportado pelo hardware e pelos `FUSE`s (ex.: AFBC/P010).
   - Validar compatibilidade entre stream e buffer: entrelaçado × progressivo, bitdepth (8-bit vs 10-bit) e subamostragem de croma (4:2:0 vs 4:2:2) em AFBC.
   - Rejeitar espelhamento em decode e rotação em saída AFBC.
   - Exigir que planos válidos residam na região `FRAMEBUF` e que planos não usados tenham endereço `0`.
   - Exigir que `stride` comporte `max_frame_width`; se o buffer for insuficiente para o quadro atual, devolvê-lo marcado com `MVE_BUFFER_FRAME_FLAG_REJECTED`.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
