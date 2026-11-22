<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Protocolo MVE 3.5 — Opções de Sessão (`SET_OPTION`)

Este documento lista todos os índices, estruturas de payload, enumerações e bitmasks suportados pela requisição `MVE_REQUEST_CODE_SET_OPTION` (`1010`), conforme o contrato público `mve_protocol_def.h` (`[P]`).

---

## 1. Formato Geral e Semântica (`[P]`)

```c
struct mve_request_set_option {
    uint32_t index; /* identificador da opção (1..69) */
    union {
        uint32_t arg;
        /* ou uma das estruturas tipadas descritas na §3 */
    } data;
};
```

### Regras de Processamento pelo Firmware

1. **Estado exigido:** Somente aceito quando a sessão está em `MVE_STATE_STOPPED` (`0`). Em `RUNNING`, responder `MVE_RESPONSE_CODE_SET_OPTION_FAIL` (`2017`).
2. **Confirmação:**
   - Sucesso → enviar `MVE_RESPONSE_CODE_SET_OPTION_CONFIRM` (`2003`) contendo `{ uint32_t index }`.
   - Parâmetro inválido, fora de faixa ou não aplicável ao codec → enviar `MVE_RESPONSE_CODE_SET_OPTION_FAIL` (`2017`) contendo `{ uint32_t index; char message[128] }`. **Não é erro fatal:** a sessão permanece ativa e a opção não tem efeito.

---

## 2. Catálogo Completo de Índices (`MVE_SET_OPT_INDEX_*`) (`[P]`)

### 2.1 Opções Comuns e de Decodificação

| Índice | Nome (`MVE_SET_OPT_INDEX_*`) | Escopo | Campo em `data` | Valores / Default |
|---:|---|---|---|---|
| `1` | `NALU_FORMAT` | Dec/Enc | `arg` | `1`=START_CODES, `2`=ONE_NALU_PER_BUFFER, `4`=1B_LEN, `8`=2B_LEN, `16`=4B_LEN, `32`=MBINFO (debug) |
| `2` | `STREAM_ESCAPING` | Dec/Enc | `arg` | `1`=habilitado (default), `0`=desabilitado |
| `3` | `PROFILE_LEVEL` | Dec/Enc | `profile_level` | `{ uint16_t profile; uint16_t level; }` (ver §3.1) |
| `4` | `HOST_PROTOCOL_PRINTS` | Comum | `arg` | `1`=habilita logs de protocolo, `0`=desabilitado (default) |
| `5` | `PROFILING` | Comum | `arg` | `1`=habilita `EVENT(PROCESSED)`/profiling, `0`=desabilitado (default) |
| `6` | `DISABLE_FEATURES` | Comum | `arg` | Bitmask `MVE_OPT_DISABLE_FEATURE_*` (ver §3.2) |
| `7` | `IGNORE_STREAM_HEADERS` | Decode | `arg` | `1`=ignora headers do stream, `0`=normal (default) |
| `8` | `FRAME_REORDERING` | Decode | `arg` | `1`=reordena saída no DPB (default), `0`=desabilitado |
| `9` | `INTBUF_SIZE` | Decode | `arg` | Limite sugerido em bytes para alocação de `IntBuf` |
| `42` | `WATCHDOG_TIMEOUT` | Comum | `arg` | Timeout do watchdog interno; `0` = desabilita watchdog |
| `57` | `DEC_DOWNSCALE` | Decode | `downscaled_frame` | `{ uint16_t width; uint16_t height; }` — downscale arbitrário (DSL) |
| `69` | `DEC_DOWNSCALE_POS_MODE` | Decode | `dsl_pos` | `{ uint32_t mode; }` — modo de posicionamento de amostragem DSL |

### 2.2 Opções de Codificação (Encode)

| Índice | Nome (`MVE_SET_OPT_INDEX_*`) | Codec | Campo em `data` | Descrição |
|---:|---|---|---|---|
| `16` | `ENC_P_FRAMES` | Enc | `arg` | Número de P-frames (`nPFrames`) entre I-frames |
| `17` | `ENC_B_FRAMES` | Enc | `arg` | Número de B-frames consecutivos |
| `18` | `GOP_TYPE` | Enc | `arg` | `1`=BIDIRECTIONAL, `2`=LOW_DELAY, `3`=PYRAMID |
| `19` | `INTRA_MB_REFRESH` | Enc | `arg` | Parâmetro de *intra macroblock refresh* |
| `20` | `ENC_CONSTR_IPRED` | Enc | `arg` | `0`/`1`: *constrained intra prediction* |
| `21` | `ENC_ENTROPY_SYNC` | HEVC Enc | `arg` | `0`/`1`: *Wavefront Parallel Processing (WPP)* |
| `22` | `ENC_TEMPORAL_MVP` | HEVC Enc | `arg` | `0`/`1`: predição temporal de MV |
| `23` | `TILES` | HEVC/VP9 | `tiles` | `{ uint16_t tile_rows; uint16_t tile_cols; }` (HEVC: `tile_cols=0`; VP9: potências de 2) |
| `24` | `ENC_MIN_LUMA_CB_SIZE` | HEVC Enc | `arg` | `8` (8×8) ou `16` (16×16) |
| `25` | `ENC_MB_TYPE_ENABLE` | Enc | `arg` | Bitmask `MVE_MBTYPE_*` a habilitar (ver §3.3) |
| `26` | `ENC_MB_TYPE_DISABLE` | Enc | `arg` | Bitmask `MVE_MBTYPE_*` a desabilitar (ver §3.3) |
| `27` | `ENC_H264_CABAC` | H.264 Enc | `arg` | `0`=CAVLC, `1`=CABAC (default `1`) |
| `28` | `ENC_SLICE_SPACING` | Enc | `arg` | Número sugerido de CTUs/MBs por slice |
| `30` | `ENC_VP9_PROB_UPDATE` | VP9 Enc | `arg` | `0`=DISABLED, `1`=IMPLICIT, `2`=EXPLICIT |
| `31` | `RESYNC_INTERVAL` | JPEG Enc | `arg` | *Restart interval* (`nRestartInterval`) |
| `32` | `HUFFMAN_TABLE` | JPEG Enc | `huffman_table` | Tabela Huffman customizada (ver §3.4) |
| `33` | `QUANT_TABLE` | JPEG Enc | `quant_table` | Matriz de quantização 8×8 (ver §3.4) |
| `34` | `ENC_EXPOSE_REF_FRAMES` | Enc (debug) | `arg` | `0`/`1`: emite `MVE_EVENT_REF_FRAME` (default `0`) |
| `35` | `MBINFO_OUTPUT` | Enc (debug) | `arg` | `0`/`1`: saída de MBInfo (proibido em sessão protegida) |
| `36` | `MV_SEARCH_RANGE` | Enc | `motion_vector_search_range` | `{ int32_t mv_search_range_x; int32_t mv_search_range_y; }` |
| `38` | `ENC_STREAM_BITDEPTH` | Enc | `bitdepth` | `{ uint16_t luma_bitdepth; uint16_t chroma_bitdepth; }` |
| `39` | `ENC_STREAM_CHROMA_FORMAT` | Enc | `arg` | Formato de croma do stream codificado (`MVE_CHROMA_FORMAT_*`) |
| `40` | `ENC_RGB_TO_YUV_MODE` | Enc | `arg` | `0`=BT601_STUDIO, `1`=BT601_FULL, `2`=BT709_STUDIO, `3`=BT709_FULL |
| `41` | `ENC_BANDWIDTH_LIMIT` | Enc | `arg` | Limite máximo de largura de banda definido pelo host |
| `43` | `ENC_CABAC_INIT_IDC` | H.264/HEVC | `arg` | `0, 1, 2` para H.264; `0, 1` para HEVC |
| `44` | `ENC_ADPTIVE_QUANTISATION` | H.264/HEVC | `arg` | Controle de quantização adaptativa |
| `45` | `QP_DELTA_I_P` | Enc | `arg` | Delta de QP entre I e P |
| `46` | `QP_DELTA_I_B_REF` | Enc | `arg` | Delta de QP entre I e B de referência |
| `47` | `QP_DELTA_I_B_NONREF` | Enc | `arg` | Delta de QP entre I e B não-referência |
| `48` | `CB_QP_OFFSET` | Enc | `arg` | Offset de QP do plano Cb |
| `49` | `CR_QP_OFFSET` | Enc | `arg` | Offset de QP do plano Cr |
| `50` | `LAMBDA_SCALE` | Enc | `lambda_scale` | 8 fatores `uint16_t` em Q8 (`0x0100` = 1.0) para I/P/B_ref/B_nonref e raízes |
| `51` | `ENC_MAX_NUM_CORES` | Enc | `arg` | Limite máximo de cores para encode |
| `56` | `ENC_FIXED_QP` | Enc | `arg` | QP fixo |
| `58` | `ENC_SAO_PENALTY` | HEVC Enc | `arg` | Penalidade SAO |
| `59` | `ENC_SAO_EN` | HEVC Enc | `arg` | Reservado / não usar (preferir 60 e 61) |
| `60` | `ENC_SAO_LUMA_EN` | HEVC Enc | `arg` | `0`/`1`: habilita SAO em Luma |
| `61` | `ENC_SAO_CHROMA_EN` | HEVC Enc | `arg` | `0`/`1`: habilita SAO em Croma |
| `62`–`65` | `ENC_CROP_RARAM_{LEFT,RIGHT,TOP,BOTTOM}` | Enc | `arg` | Parâmetros de *conformance crop* na codificação |
| `66` | `ENC_LTR_MODE` | Enc | `arg` | Modo de *Long Term Reference* |
| `67` | `ENC_LTR_PERIOD` | Enc | `arg` | Período de quadros LTR |

---

## 3. Detalhamento de Bitmasks e Estruturas Tipadas (`[P]`)

### 3.1 `profile_level` (Índice `3`)

| Codec | Perfis (`profile`) | Níveis (`level`) |
|---|---|---|
| **H.264** | `1`=Baseline, `2`=Main, `3`=High, `4`=High 10 | `1`=1, `2`=1b, `3`=1.1, `4`=1.2, `5`=1.3, `6`=2, `7`=2.1, `8`=2.2, `9`=3, `10`=3.1, `11`=3.2, `12`=4, `13`=4.1, `14`=4.2, `15`=5, `16`=5.1, `17`=5.2, `18`=6, `19`=6.1, `20`=6.2; `32..287` = `32 + level_idc` literal |
| **HEVC** | `1`=Main, `2`=Main Still, `3`=Main Intra, `4`=Main 10 | Ímpares `1,3,5,7,9,11,13,15,17,19,21,23,25` = Main Tier (`1, 2, 2.1, 3, 3.1, 4, 4.1, 5, 5.1, 5.2, 6, 6.1, 6.2`); pares `2..26` = High Tier correspondente |
| **VC-1** | `1`=Simple, `2`=Main, `3`=Advanced | — |
| **VP8** | `1`=Main | — |

### 3.2 Bitmask `MVE_OPT_DISABLE_FEATURE_*` (Índice `6`)

| Bit / Máscara | Macro | Recurso Desabilitado |
|---|---|---|
| `0x0000_0001` | `AFBC_COMP` | Compressão AFBC no VDMA |
| `0x0000_0002` | `REF_CACHE` | Cache do subsistema REF |
| `0x0000_0004` | `DEBLOCK` | Filtro de deblocking |
| `0x0000_0008` | `SAO` | Filtro SAO (HEVC) |
| `0x0000_0020` | `PIC_OUTPUT` | Remoção da escrita de saída de imagem |
| `0x0000_0040` | `PIPE` | Estágio de pipe (opera em modo *parser-only*) |
| `0x0000_0080` | `SLEEP` | *Clock gating* interno (`SYSCTRL.SLEEP`) |
| `0x0000_0100` | `AFBC_LEGACY_REF` | Habilita formato AFBC tiled em buffers de referência (ignorado na saída AFBC de decode) |
| `0x0000_0400` | `REF_PICS` | Força uso de referências estáticas 16×16 |
| `0x0000_0800` | `CHNG_RECT_WA` | Desabilita workaround de *change rectangles* |
| `0x0000_1000` | `REFSZ_LIMIT` | Desabilita limite de banda `REFSZ` |

### 3.3 Bitmask `MVE_MBTYPE_*` (Índices `25` e `26`)

- Inter: `4x4=0x0001`, `4x8=0x0002`, `8x4=0x0004`, `8x8=0x0008`, `8x16=0x0010`, `16x8=0x0020`, `16x16=0x0040`, `PSKIP=0x0080`, `16x32=0x1000`, `32x16=0x2000`, `32x32=0x4000`.
- Intra: `I4x4=0x0100`, `I8x8=0x0200`, `I16x16=0x0400`, `I32x32=0x0800`.

### 3.4 Estruturas de Tabelas JPEG (Índices `32` e `33`)

```c
struct {
    uint32_t type; /* 1=DC_LUMA, 2=AC_LUMA, 3=DC_CHROMA, 4=AC_CHROMA */
    uint8_t  number_of_huffman_of_code_length[16];
    uint8_t  table[162]; /* 12 usados para DC, 162 para AC */
} huffman_table;

struct {
    uint32_t type; /* 1=LUMA, 2=CHROMA */
    uint8_t  matrix[64];
} quant_table;
```

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
