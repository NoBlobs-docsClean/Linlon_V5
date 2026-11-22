<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Subsistemas Internos de Vídeo e Pipeline de Hardware

Este documento descreve os blocos fixos de aceleração de vídeo presentes em cada core da VPU Linlon V5, o papel arquitetural de cada bloco, os fluxos de dados de decodificação e codificação, o modelo de buffers intermediários (`IntBuf`), o gerenciamento do DPB e os requisitos de coordenação multi-core.

---

## 1. Catálogo dos Subsistemas de Hardware por Core (`[P][D]`)

Cada core Linlon V5 combina **5 CPUs RASC** com os seguintes blocos aceleradores fixos, cujas funções e domínios de controle estão estabelecidos na arquitetura pública Arm China Linlon V5/V7 e refletidos nas opções do protocolo MVE 3.5:

| Bloco | Nome Completo | Papel no Hardware | Recursos Controláveis via MVE |
|---|---|---|---|
| **BITSTREAM** | Motor de Entropia / Bitstream | Parsing e codificação de entropia acelerada em hardware: CABAC, CAVLC, decodificador booleano VP8/VP9, predição de vetores de movimento, busca de start-codes e tratamento de *emulation prevention* (*escaping*) | `NALU_FORMAT`, `STREAM_ESCAPING`, `ENC_H264_CABAC`, `ENC_CABAC_INIT_IDC`, `ENC_VP9_PROB_UPDATE`, `ENC_ENTROPY_SYNC` |
| **REF** | Subsistema de Referência e Cache | Armazena, indexa e realiza *prefetch*/cache de quadros de referência (preferencialmente em formato comprimido **AFBC**), com gerenciamento de banda e limites de tamanho | `DISABLE_FEATURE_REF_CACHE`, `DISABLE_FEATURE_AFBC_LEGACY_REF`, `DISABLE_FEATURE_REF_PICS`, `DISABLE_FEATURE_REFSZ_LIMIT` |
| **ME** | *Motion Estimation* | Busca de vetores de movimento em encode (busca hierárquica / janela configurável X×Y, seleção de partições MB/CU) | `MV_SEARCH_RANGE`, `ENC_MB_TYPE_ENABLE/DISABLE`, `ENC_MIN_LUMA_CB_SIZE` |
| **MC** | *Motion Compensation* | Interpolação sub-pixel e predição inter-quadro utilizando referências do bloco `REF` | Pipeline de decode e reconstrução de encode |
| **TRF** | *Transform & Quantization* | Transformada direta/inversa (DCT/DST/integer transforms 4×4..32×32) e quantização/dequantização | `QUANT_TABLE` (JPEG), `QP`, `CB/CR_QP_OFFSET`, `ADAPTIVE_QUANTISATION` |
| **DBL** | *Deblocking & Loop Filter* | Filtro de desblocagem em laço (H.264/HEVC/VP8/VP9/etc.) e **SAO** (*Sample Adaptive Offset*, HEVC) | `DISABLE_FEATURE_DEBLOCK`, `DISABLE_FEATURE_SAO`, `ENC_SAO_PENALTY/LUMA_EN/CHROMA_EN` |
| **SDMA** | *System DMA* | Transferência de dados estruturados entre DRAM e SRAM local: `mbinfo`, coeficientes residuais, linhas de contexto (*row buffers*) e metadados de stripes | Buffers intermediários (`IntBuf`), `MBINFO_OUTPUT` |
| **VDMA** | *Video DMA* | Transferência de stripes de pixels entre DRAM (`FRAMEBUF`) e pipeline: compressão/descompressão AFBC, rotação (90°/180°/270°), espelhamento, conversão RGB→YUV e **downscale flexível** (DSL) | `DISABLE_FEATURE_AFBC_COMP`, `ROTATION_*`, `MIRROR_*`, `DEC_DOWNSCALE`, `ENC_RGB_TO_YUV_MODE` |
| **SYSCTRL** | Controle de Sistema do Core | Revisão de hardware do core, controle de *clock gating* / `SLEEP` interno e temporização/watchdog interno | `DISABLE_FEATURE_SLEEP`, `WATCHDOG_TIMEOUT` |
| **MMU** | Unidade de Gerenciamento de Memória | Tradução VA 32b → PA 40b por LSID e detecção de falhas de acesso (`MMU ABORT`) | Controlada pelo host (`MMU_CTRL`, `FLUSH_ALL`) |

---

## 2. Fluxo de Dados — Decodificação (`[P][D]`)

A decodificação é desacoplada em **dois estágios macro** (o que permite que um core ou CPU RASC faça o parsing do quadro *N+1* enquanto outro reconstrói o quadro *N*):

```text
  Bitstream em DDR (região PROTECTED)
       │
       ▼  (DMA de bitstream)
  ┌────────────────────────── ESTÁGIO 1: PARSE ──────────────────────────┐
  │  BITSTREAM (CABAC / CAVLC / VP8-VP9 bool / VLC) + CPU RASC           │
  │   • Decodifica headers (SPS/PPS/VPS/Slice)                           │
  │   • Valida resolução, perfil, bitdepth, croma                        │
  │   • Extrai símbolos de entropia → produz mbinfo + residuais          │
  └──────────────────────────────────┬───────────────────────────────────┘
                                     │  SDMA
                                     ▼
                    IntBuf (DDR — alocado via RPC MEM_ALLOC)
                                     │  SDMA
                                     ▼
  ┌────────────────────────── ESTÁGIO 2: PIPE ───────────────────────────┐
  │  Reconstrução por Stripes / Macroblocks / CTUs:                      │
  │                                                                      │
  │   REF (referências AFBC) ──► MC ──► TRF (IQ/IT) ──► DBL (+ SAO)      │
  │             ▲                                            │           │
  │             └─────── gravação de quadro de referência ◄──┤           │
  │                                                          ▼           │
  │                       VDMA (planar / AFBC / rotação / downscale)     │
  └──────────────────────────────────┬───────────────────────────────────┘
                                     │  AXI 128-bit
                                     ▼
                   Framebuffer de saída em DDR (região FRAMEBUF)
```

### Requisitos Comportamentais da Decodificação

1. **Desacoplamento Parse ↔ Pipe (`IntBuf`):**
   - O estágio de *parse* grava `mbinfo` e coeficientes residuais em buffers intermediários (`IntBuf`) alocados dinamicamente via RPC (`MVE_RPC_FUNCTION_MEM_ALLOC`, normalmente na região `PROTECTED`).
   - A opção `MVE_SET_OPT_INDEX_INTBUF_SIZE` (`9`) sugere um limite de tamanho para o alocador de `IntBuf`.
   - A flag `MVE_OPT_DISABLE_FEATURE_PIPE` (`0x40`) permite operar em modo *parser-only* (diagnóstico).
2. **Gerenciamento de DPB e Reordenação:**
   - O firmware gerencia o **DPB (*Decoded Picture Buffer*)**: ordem de exibição (*display order*), marcação de quadros de referência (`is_ref`), quadros *decode-only* (`MVE_BUFFER_FRAME_FLAG_DECODE_ONLY`, ex.: frames invisíveis VP9 ou pré-roll) e drenagem em `EOS` ou troca de sequência.
   - `MVE_SET_OPT_INDEX_FRAME_REORDERING` (`8`, default `1`) habilita ou desabilita reordenação de saída no DPB.
   - O firmware pode informar ao host quantos buffers de saída estão retidos no DPB para reordenação via `MVE_BUFFER_PARAM_TYPE_DPB_HELD_FRAMES` (`19`).
3. **Referências AFBC e Bancos Privados:**
   - Quando a saída solicitada pelo host é **AFBC**, os próprios buffers de saída fornecidos pelo host podem atuar como quadros de referência (o firmware devolve o buffer marcado com `MVE_BUFFER_FRAME_FLAG_REF_FRAME`; o host não deve sobrescrevê-lo e deve re-enfileirá-lo após exibição, ou enviar `MVE_REQUEST_CODE_RELEASE_REF_FRAME` e aguardar `MVE_RESPONSE_CODE_REF_FRAME_UNUSED`).
   - Quando a saída solicitada pelo host é **planar** (ex.: NV12/I420), o hardware ainda precisa de referências internas (em AFBC); o firmware aloca um banco privado de referências via RPC (`MVE_MEM_REGION_OUTBUF`/`FRAMEBUF`) — motivo pelo qual `sequence_parameters.num_buffers_afbc` é maior que `num_buffers_planar` (`[P]`).
4. **Mudança Dinâmica de Sequência:**
   - Ao decodificar o primeiro cabeçalho de sequência (ou detectar mudança de resolução, interlace, formato de croma ou profundidade de bits), o firmware deve emitir `MVE_RESPONSE_CODE_FRAME_ALLOC_PARAM` seguido de `MVE_RESPONSE_CODE_SEQUENCE_PARAMETERS` e entrar em espera até o host enviar `MVE_REQUEST_CODE_OUTPUT_FLUSH`, drenar todos os buffers de saída antigos, emitir `MVE_RESPONSE_CODE_OUTPUT_FLUSHED` e receber os novos buffers realocados (`[P][O]`).

---

## 3. Fluxo de Dados — Codificação (`[P][D]`)

```text
  Frame de entrada em DDR (YUV planar / semi-planar / AFBC / RGB)
       │
       ▼  VDMA (opcional: conversão RGB→YUV, crop, rotação, mirror)
  ┌────────────────────────── PIPELINE DE ENCODE ────────────────────────┐
  │                                                                      │
  │   REF (quadros reconstruídos / LTR) ◄──────────────────┐             │
  │             │                                          │             │
  │             ▼                                          │             │
  │   ME (busca de movimento) ──► MC + Intra Pred          │             │
  │                                    │                   │             │
  │                                    ▼                   │             │
  │                           TRF (T/Q + IQ/IT) ──► DBL (+ SAO)          │
  │                                    │                                 │
  │                                    ▼                                 │
  │                     BITSTREAM (CABAC / CAVLC / VP8-VP9 bool / JPEG)  │
  └──────────────────────────────────┬───────────────────────────────────┘
                                     │  DMA de bitstream
                                     ▼
                  Buffer de bitstream de saída (região PROTECTED)
```

### Requisitos Comportamentais da Codificação

- **Controle de Taxa e QP:** Suporta QP fixo por tipo de quadro (`QP_I`, `QP_P`, `QP_B`), regiões de interesse (`MVE_BUFFER_PARAM_TYPE_REGIONS`, até 16 regiões de macroblocks), configuração por bloco/quad (`MVE_BUFFER_GENERAL_TYPE_BLOCK_CONFIGS`), quantização adaptativa (`ADPTIVE_QUANTISATION`), escalas de lambda em ponto fixo Q8 (`LAMBDA_SCALE`) e modos de *rate control* (`OFF`, `STANDARD`, `VARIABLE`, `CONSTANT`, `C_VARIABLE`, com `qp_min`/`qp_max` e tamanho de buffer HRD).
- **Estrutura de GOP e Referências:** Suporta GOP bidirecional, low-delay e piramidal (`GOP_TYPE`), intra refresh (`INTRA_MB_REFRESH`), *Long Term Reference* (`ENC_LTR_MODE`/`PERIOD`), reset de GOP (`MVE_BUFFER_PARAM_TYPE_GOP_RESET`) e retângulos de mudança (`CHANGE_RECTANGLES`, até 2 retângulos) para economia de banda.
- **Paralelismo:** Tiles configuráveis (`TILES`: linhas/colunas), `ENC_ENTROPY_SYNC` (WPP em HEVC), `ENC_SLICE_SPACING` e escalonamento multi-core (`ENC_MAX_NUM_CORES`).

---

## 4. Coordenação Multi-Core e Gerenciamento de Banda (`[P][D]`)

Quando um job é submetido com `cores > 1` (ex.: `cores = 2` no K1 ou `cores = 4` no K3):

1. **Papel Master / Slave entre Cores:**
   - Um dos cores físicos atua como coordenador de sessão (dono das filas MVE e da distribuição de quadros/slices) e os demais atuam como *slaves* executando stripes/quadros atribuídos.
   - O estado compartilhado da sessão reside na região **`master_rw`** (páginas físicas comuns mapeadas no mesmo offset virtual em todas as instâncias de core).
2. **Gerenciador de Banda (*Bandwidth Manager*):**
   - O hardware/firmware pode arbitrar acesso à DDR para limitar picos de banda (`ENC_BANDWIDTH_LIMIT` em encode; `MVE_JOB_FLAG_DISABLE_BNDMGR = 0x01` no `JOB` desabilita o gerenciador de banda).
3. **Contabilidade e Profiling (`MVE_SET_OPT_INDEX_PROFILING`):**
   - Quando habilitado, o firmware coleta timestamps de início/fim/idle de *parse* e *pipe*, `parser_coreid`, `pipe_coreid`, bits de bitstream, tamanho de `IntBuf`, memória total alocada, bytes lidos/escritos no barramento e bytes AFBC, reportando ao host via `MVE_EVENT_PROCESSED` (`3`).

---

## 5. Estado de Especificação dos Registradores de Vídeo

Os nomes lógicos e comportamentos acima estão estabelecidos pelo contrato público `[P]` e observações `[O]`. O mapa de registradores internos **campo por campo** de `BITSTREAM`, `REF`, `ME`, `MC`, `TRF`, `DBL`, `SDMA` e `VDMA` na faixa `0xFF00_0000`–`0xFF2F_8FFF` permanece em aberto — ver [internal-mmio-map.md](internal-mmio-map.md) e [observed-sequences-and-gaps.md](observed-sequences-and-gaps.md).

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
