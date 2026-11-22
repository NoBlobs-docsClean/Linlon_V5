<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Visão Geral — VPU Arm China Linlon V5 (SpacemiT K1 e K3)

Este documento apresenta a arquitetura de alto nível da VPU integrada aos SoCs **SpacemiT Key Stone K1** e **SpacemiT K3**, a relação entre as variantes de hardware e a divisão de responsabilidades entre o driver hospedeiro (Linux) e o firmware executado nas CPUs embarcadas.

---

## 1. Identificação do IP de Hardware (`[P][O]`)

| Atributo | Valor | Proveniência |
|---|---|---|
| **Fabricante do IP** | Arm Technology (China) Co., Ltd. | `[P]` |
| **Nome comercial / Linha** | **Linlon V5** (família de driver `MVE_v52_v76`, derivada da linhagem Arm Mali-V52/V76) | `[P][O]` |
| **`HARDWARE_ID` (offset `0x00`)** | **`0x56635002`** (família `0x5663`, revisão `0x50` = 80, patch `0x02`) — idêntico no K1 e no K3 | `[O]` |
| **`compatible` (Device Tree)** | `"arm china,linlon-v5"` (K1 e K3) | `[P][O]` |
| **Diretório de firmware esperado** | `linlon-v52_v76-80-2/<codec><dec\|enc>.fwb` | `[P]` |
| **Protocolo Host ↔ Firmware** | **MVE 3.5** (cabeçalho `protocol_major = 3`, `protocol_minor = 5`) | `[P][O]` |
| **Processador embarcado** | **5 × CPUs RASC** por core de VPU (RISC de 32 bits, pipeline de 5 estágios, little-endian) | `[P]` |
| **Contextos de sessão (LSID)** | **4 LSIDs** (`NLSID = 4`) no K1 e no K3 | `[O]` |
| **Fuses (`FUSE`, offset `0x34`)** | `0x00000000` (todos os recursos habilitados: AFBC, RealVideo, VP8/VP9, HEVC) | `[O]` |
| **Interfaces de barramento** | AXI 128 bits (dados/DDR) + APB3 (registradores de controle do host) | `[P]` |
| **Compressão de framebuffer** | Suporte nativo a **AFBC** (Arm Frame Buffer Compression) 8-bit e 10-bit | `[P]` |

> **Nota crítica sobre nomenclatura (`[P][O]`):** O driver Linux mapeia tanto a família `0x5662` quanto a `0x5663` para o mesmo identificador interno `MVE_v52_v76` (`0x5276`). **Nem o K1 nem o K3 são V76 (classe 8K):** ambos reportam `HARDWARE_ID = 0x56635002` e `compatible = "arm china,linlon-v5"`, sendo configurações classe **4K** da mesma geração Linlon V5.

---

## 2. Comparação Estrutural: SpacemiT K1 × SpacemiT K3 (`[P][O]`)

Ambos os SoCs instanciam o **mesmo IP de VPU**, diferindo na contagem de cores, frequência de clock e valor do registrador de topologia `CONFIG`:

| Parâmetro | SpacemiT K1 | SpacemiT K3 | Impacto no Firmware |
|---|---|---|---|
| `HARDWARE_ID` | `0x56635002` | `0x56635002` | Nenhum (mesmo IP e revisão) |
| `NCORES` | **2** | **4** | Transparente: o host instancia o mesmo binário por core e informa `cores` na mensagem `JOB` |
| `NLSID` | 4 | 4 | Nenhum |
| `CORELSID` (em repouso) | `0x000000FF` (2 nibbles) | `0x0000FFFF` (4 nibbles) | Gerenciado apenas pelo driver host |
| `CONFIG` | `0xFFFF6402` | `0xFFFF7404` | Lido apenas pelo host (diferença de topologia) |
| Clock operacional no probe | **819,2 MHz** | **1 GHz** | Nenhum: o firmware não controla o clock |
| Base MMIO host | `0xC050_0000` | `0xC050_0000` | Lado host apenas |
| Janela `reg` no DT | `64 KiB` (`0x10000`) | `8 KiB` (`0x2000`) | Lado host apenas |
| IRQ | `74` (`0x4A`) | `74` (`0x4A`, level high) | Lado host apenas |
| Capacidade nominal (Decode) | 4K@60 (H.264/HEVC) | 4K@180 | Escala linearmente com `NCORES` × clock |
| Capacidade nominal (Encode) | 4K@30 (H.264/HEVC) | 4K@90 | Escala linearmente com `NCORES` × clock |

**Regra arquitetural fundamental (`[P][O]`):** Um único binário de firmware `.fwb` por `(codec, direção)` deve funcionar **sem recompilação** tanto no K1 (2 cores) quanto no K3 (4 cores), desde que:
1. Utilize endereçamento **PC-relativo** (`lpc`) para acessar seu texto e BSS privado (pois cada instância de core é mapeada em `core * 0x0100_0000`).
2. Aceite dinamicamente o número de cores (`1..NCORES`) recebido na mensagem `MVE_REQUEST_CODE_JOB` (respeitando que o número de cores é fixo durante um job).
3. Não dependa de frequências absolutas de clock para correção funcional (apenas para contadores de tempo/watchdog).

---

## 3. Arquitetura Interna de um Core Linlon V5 (`[P][D]`)

Cada core físico da VPU não é um acelerador monolítico hardwired, nem um DSP de pixels em software: é um **complexo heterogêneo controlado por firmware**, onde **5 microcontroladores RISC (CPUs RASC)** administram blocos fixos de processamento de vídeo e DMAs.

**Princípio central de projeto (`[P]`):** *"As CPUs RASC nunca tocam pixels diretamente — elas administram os subsistemas de hardware."*

```text
┌────────────────────────────── VPU Linlon V5 (Top-Level) ──────────────────────────────┐
│  Interface APB (0xC050_0000) ──► Escalonador Global (JOBQUEUE, CORELSID, 4 × LSID)    │
│  Interface AXI 128-bit       ◄── Árbitro de Barramento ◄── MMU (por LSID, 2 níveis)   │
│                                                                                       │
│  ┌────────────────────── Core 0 ──────────────────────┐  ┌──── Core 1 (..3 no K3) ──┐ │
│  │  5 × CPUs RASC (32-bit, 5-stage RISC)              │  │  5 × CPUs RASC           │ │
│  │    [RASC0 Master] [RASC1] [RASC2] [RASC3] [RASC4]  │  │  REF / ME / MC / TRF /   │ │
│  │                                                    │  │  DBL / BITSTREAM /       │ │
│  │  Subsistemas de Vídeo:                             │  │  SDMA / VDMA / SRAM      │ │
│  │    BITSTREAM │ REF (AFBC) │ ME │ MC │ TRF │ DBL    │  │                          │ │
│  │                                                    │  │                          │ │
│  │  Movimentação de Dados e Controle:                 │  │                          │ │
│  │    SDMA │ VDMA │ SYSCTRL │ MMU Local │ SRAM Local  │  │                          │ │
│  └────────────────────────────────────────────────────┘  └──────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

### Componentes de Cada Core

1. **5 CPUs RASC (32 bits):** Executam o firmware `.fwb` mapeado pela MMU. Possuem espaço MMIO interno (`0xFF00_0000`–`0xFF2F_8FFF`) para programar todos os blocos de vídeo e sinalizar interrupções ao host.
2. **6 Subsistemas de Vídeo:**
   - **BITSTREAM:** Entropia (CABAC, CAVLC, decodificação booleana VP8/VP9), parsing acelerado e predição de vetores de movimento.
   - **REF:** Gerenciamento e cache de quadros de referência (incluindo AFBC).
   - **ME:** Estimação de movimento (*Motion Estimation*, usado em encode).
   - **MC:** Compensação de movimento (*Motion Compensation*).
   - **TRF:** Transformada direta/inversa e quantização/dequantização.
   - **DBL:** Filtro de *deblocking* (e SAO em HEVC).
3. **2 Motores DMA:**
   - **SDMA:** DMA de sistema para metadados, linhas/rows, `mbinfo` e buffers intermediários.
   - **VDMA:** DMA de vídeo para stripes de pixels (planar, semi-planar, packed RGB/YUV e AFBC).
4. **MMU por Sessão (LSID):** Tradução VA 32 bits → PA 40 bits em 2 níveis (páginas de 4 KiB), isolando completamente cada sessão.
5. **Escalonador Multi-Core / Multi-Sessão:** Permite até 4 sessões residentes em hardware (LSIDs 0–3) disputando os cores físicos via `JOBQUEUE`.

---

## 4. Fronteira de Responsabilidades: Host × Firmware (`[P]`)

A arquitetura MVE 3.5 impõe uma separação estrita de camadas. O firmware **nunca conhece V4L2, fourccs ou políticas de energia do SoC**; o host **nunca faz parsing de bitstream, gestão de DPB ou programação de blocos de vídeo**.

| Responsabilidade | Lado Host (Driver Linux `mvx`) | Lado Firmware (CPUs RASC) |
|---|---|---|
| API de usuário | V4L2 Mem2Mem (`/dev/video*`), controles V4L2, `dmabuf` | Não visível (vê apenas protocolo MVE 3.5) |
| Clock, Reset, Power, DVFS | Configura `CLK_VPU`, `RESET_VPU`, `PMU`, `RESET`/`CLKFORCE` | Não tem acesso ao CCU/PMU |
| Escalonamento entre sessões | Mapeia LSIDs (`0..3`), escreve `JOBQUEUE`, `CTRL`, `SCHED`, envia `SWITCH` | Executa até ficar ocioso ou receber `SWITCH`; salva contexto e responde `SWITCHED_OUT` |
| MMU e Memória Física | Aloca páginas, preenche tabelas L1/L2, aplica `phys_cpu2vpu`, escreve `MMU_CTRL` e `FLUSH_ALL` | Usa exclusivamente VAs de 32 bits; solicita memória adicional ao host via RPC (`MEM_ALLOC`/`RESIZE`/`FREE`) |
| Carga do Firmware | Lê `.fwb`, valida header, mapeia texto (RO/X) + BSS privado + `master_rw` compartilhado em cada instância | Executa a partir de `VA 0x1000` (trampolim no offset 0), inicializa pilha PC-relativa e workers |
| Parsing de Bitstream | Envia buffers brutos (`mve_buffer_bitstream`) com flags (`EOS`, `CODECCONFIG`, etc.) | Decodifica cabeçalhos (SPS/PPS/VPS/slices), detecta mudanças de sequência, emite `FRAME_ALLOC_PARAM` e `SEQUENCE_PARAMETERS` |
| Gerenciamento de DPB e Referências | Fornece buffers de frame (`mve_buffer_frame`), respeita flag `REF_FRAME`, pode enviar `RELEASE_REF_FRAME` | Decide reordenação, referências ativas, bancos privados AFBC, descarte (`DECODE_ONLY`) e `REF_FRAME_UNUSED` |
| Pipeline de Vídeo | — | Programa e sincroniza `BITSTREAM`, `REF`, `ME`, `MC`, `TRF`, `DBL`, `SDMA`, `VDMA` em stripes |
| Diagnóstico e Erros | Lê `IRQVE`/`LIRQVE`, drena filas, lê `ram_print`, mantém watchdog de sessão | Emite `ERROR`, `EVENT` (`PROCESSED`, `STREAM_CORRUPT`, etc.), escreve em `ram_print` e executa `DUMP` |

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
