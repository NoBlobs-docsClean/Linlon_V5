<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Codecs Suportados, Seleção de Firmware e Matriz de Capacidades

Este documento consolida todos os codecs de vídeo e imagem suportados pelo hardware Arm China Linlon V5 (`MVE_v52_v76`, `0x56635002`) no SpacemiT K1 e no SpacemiT K3, o mapeamento exato de arquivos `.fwb` esperado pelo driver Linux, a matriz de formatos de pixel e as restrições operacionais por codec.

---

## 1. Mapeamento de Arquivos `.fwb` Esperados pelo Driver (`[P]`)

O driver `mvx` carrega um binário independente por `(codec, direção)` a partir de `linlon-v52_v76-80-2/<codec><dec|enc>.fwb`. Ao todo existem **16 combinações de arquivos** (11 blobs de decode cobrindo 12 padrões + 5 blobs de encode):

| Padrão / Codec | Arquivo `.fwb` (Decode) | Arquivo `.fwb` (Encode) | Observação de Contrato |
|---|---|---|---|
| **H.264 / AVC** | `h264dec.fwb` | `h264enc.fwb` | Perfis BP / MP / HP / High 10 (MVE 3.x) |
| **HEVC / H.265** | `hevcdec.fwb` | `hevcenc.fwb` | Main / Main 10 / Main Still / Main Intra; sujeito a `FUSE_DISABLE_HEVC` |
| **VP8** | `vp8dec.fwb` | `vp8enc.fwb` | Sujeito a `FUSE_DISABLE_VPX` |
| **VP9** | `vp9dec.fwb` | `vp9enc.fwb` | Profile 0 (8-bit) e Profile 2 (10-bit); sujeito a `FUSE_DISABLE_VPX` |
| **JPEG / MJPEG** | `jpegdec.fwb` | `jpegenc.fwb` | Baseline sequential até 8192×8192 (melhor candidato para Marco M4!) |
| **MPEG-2** | `mpeg2dec.fwb` | — | Main Profile (somente decode) |
| **MPEG-4** | `mpeg4dec.fwb` | — | Simple / Advanced Simple Profile (somente decode) |
| **H.263** | **`mpeg4dec.fwb`** | — | **Sem blob dedicado:** o driver mapeia `MVX_FORMAT_H263` para `mpeg4dec.fwb` (Profile 0) |
| **VC-1** | `vc1dec.fwb` | — | Simple / Main / Advanced Profile (suporta `range_mapping_enabled`) |
| **RealVideo 8/9/10** | `rvdec.fwb` | — | Sujeito a `FUSE_DISABLE_REAL` |
| **AVS / AVS+** | `avsdec.fwb` | — | Somente decode |
| **AVS2** | `avs2dec.fwb` | — | Somente decode |

---

## 2. Matriz de Capacidades e Metas de Desempenho (K1 × K3) (`[P][O]`)

### 2.1 Decodificação

| Codec | Perfis | Resolução Máxima | K1 (2 cores @ 819 MHz) | K3 (4 cores @ 1 GHz) |
|---|---|---|---|---|
| HEVC / H.265 | Main / Main 10 | 4096×4096 | 4K@60 | até 4K@180 |
| H.264 / AVC | BP / MP / HP / High 10 | 4096×4096 | 4K@60 | até 4K@180 |
| VP9 | Profile 0 / Profile 2 (10-bit) | 4096×4096 | 4K@30 | escala por 4 cores |
| VP8 | Main | 2048×2048 | 1080p@60 | escala por 4 cores |
| JPEG / MJPEG | Baseline sequential | 8192×8192 | 4K@30 | escala por 4 cores |
| MPEG-2 | Main Profile | 4096×4096 | 1080p@60 | escala por 4 cores |
| MPEG-4 / H.263 | SP / ASP / H.263 P0 | 2048×2048 | 1080p@60 | escala por 4 cores |
| VC-1 | SP / MP / AP | 2048×4096 | 1080p@60 | escala por 4 cores |
| AVS / AVS+ / AVS2 / RV8–10 | Padrões nativos | habilitado no IP | suportado | suportado |

### 2.2 Codificação

| Codec | Perfis | Resolução Máxima | K1 (2 cores @ 819 MHz) | K3 (4 cores @ 1 GHz) |
|---|---|---|---|---|
| HEVC / H.265 | Main / Main 10 | 4096×4096 | 4K@30 | até 4K@90 |
| H.264 / AVC | BP / MP / HP / High 10 | 4096×4096 | 4K@30 | até 4K@90 |
| VP9 | Profile 0 / Profile 2 (10-bit) | 4096×4096 | 4K@30 | escala por 4 cores |
| VP8 | Main | 2048×2048 | 1080p@60 | escala por 4 cores |
| JPEG / MJPEG | Baseline sequential | 8192×8192 | 4K@30 | escala por 4 cores |

> **Desacoplamento ISP vs VPU (`[P]`):** O SpacemiT K1 possui também um codificador JPEG dedicado no subsistema de câmera/ISP (até 23 MP). Ele é um bloco de hardware totalmente separado e não tem relação com `jpegenc.fwb`/`jpegdec.fwb` da VPU.

---

## 3. Matriz de Formatos de Pixel Suportados (`mvx_hwreg_get_formats_v52_v76`) (`[P]`)

| Formato de Pixel | Entrada (Encode) | Saída (Decode) | Restrições |
|---|:---:|:---:|---|
| `YUV420_I420` | Sim | Sim | 3 planos (Y, U, V), 8-bit |
| `YUV420_NV12` / `NV21` | Sim | Sim | 2 planos (Y + UV/VU), 8-bit |
| `YUV420_P010` | Sim | Sim | 2 planos, 10-bit em 16 bits |
| `YUV420_Y0L2` / `AQB1` | Sim | Sim | 1 plano, 10-bit packed |
| `YUV422_YUY2` / `UYVY` | Sim | Sim | 1 plano, 8-bit packed |
| `YUV422_Y210` | Sim | Sim | 1 plano, 10-bit |
| `YUV420_AFBC_8` / `10` | Sim | Sim | Desabilitado se `FUSE` bit 0 = 1 |
| `YUV422_AFBC_8` / `10` | Sim | Sim | Desabilitado se `FUSE` bit 0 = 1 |
| `RGBA/BGRA/ARGB/ABGR_8888` | **Sim** | **Não** | Conversão RGB→YUV via `ENC_RGB_TO_YUV_MODE` apenas em encode |

---

## 4. Restrições Comportamentais Específicas por Codec (`[P][D]`)

1. **H.264 Decode:**
   - Suporta formatos de empacotamento NALU via `MVE_SET_OPT_INDEX_NALU_FORMAT` (`START_CODES`, `ONE_NALU`, prefixos de 1/2/4 bytes) e *emulation prevention* (`STREAM_ESCAPING`).
   - Streams MBAFF com largura acima de 2048 pixels são rejeitados.
   - Emite `ctu_size = 16` em `FRAME_ALLOC_PARAM`.
2. **HEVC Decode / Encode:**
   - `ctu_size` pode ser 16, 32 ou 64.
   - Em encode, `TILES.tile_cols` deve ser `0` (apenas linhas de tile), `ENC_MIN_LUMA_CB_SIZE` aceita `8` ou `16`, e SAO é controlado separadamente para luma (`60`) e croma (`61`).
3. **VP9 Decode / Encode:**
   - Em encode, `TILES.tile_rows` e `TILES.tile_cols` devem ser potências de 2; atualização de probabilidades segue `ENC_VP9_PROB_UPDATE` (`0=OFF`, `1=IMPLICIT`, `2=EXPLICIT`).
   - Frames ocultos (*alt-ref* / `show_frame = 0`) utilizam `MVE_BUFFER_FRAME_FLAG_DECODE_ONLY` no DPB.
4. **JPEG Encode / Decode:**
   - Não possui DPB, referências inter-quadro nem motion compensation — ideal como **primeiro alvo real de aceleração de hardware (Marco M4)** para validar `BITSTREAM` + `TRF` + `VDMA`.
   - Em encode, aceita `RESYNC_INTERVAL` (`31`), `HUFFMAN_TABLE` (`32`) e `QUANT_TABLE` (`33`).
5. **VC-1 Advanced Profile Decode:**
   - Sinaliza `sequence_parameters.range_mapping_enabled` quando ativo: para saída planar, o próprio firmware aplica o mapeamento de faixa; para saída AFBC, envia `MVE_BUFFER_PARAM_TYPE_RANGE_MAP` (`6`) para filtragem pós-decode pelo consumidor.
6. **Downscaling Arbitrário (DSL) em Decode:**
   - Configurado via `DEC_DOWNSCALE` (`57`) e `DEC_DOWNSCALE_POS_MODE` (`69`); não é suportado para saída tiled/AFBC fora das restrições documentadas nem para entradas com dimensão inferior a 2 pixels.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
