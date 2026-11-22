<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Mapa Consolidado de Registradores e SRAMs Internas (`0xFF00_0000` – `0xFF2F_8FFF`)

No espaço virtual de 32 bits do protocolo MVE 3.x, a faixa **`0xF000_0000` – `0xFFFF_FFFF`** é reservada ao uso interno da VPU (`[P]`). As CPUs RASC acessam SRAMs internas compartilhadas, controladores de interrupção, watchdog, MMU interna, DMAs e os 6 blocos aceleradores de vídeo através da faixa **`0xFF00_0000` – `0xFF2F_8FFF`** (`[O][D]`).

> **Nota sobre construção de endereços (`[O]`):** Na ISA RASC, endereços `0xFFxxxxxx` são construídos com `movn rd, imm`, cuja semântica real no silício é **NOT bit a bit (`~imm`)** — ex.: `movn r, 0xFD4FFF` produz `0xFF02_B000`. Todos os endereços abaixo refletem o alinhamento real validado em hardware.

---

## 1. Mapa Geral do Espaço Interno do Core (`[O][D]`)

| Base | Tamanho / Faixa | Bloco de Hardware | Descrição |
|---|---|---|---|
| **`0xFF00_0000`** | On-chip SRAM | **Session SRAM** (inclui janela STP do SDMA em `0xFF00_1C00`) | SRAM compartilhada entre **todos os cores físicos** de um mesmo LSID (layout definido pelo firmware) |
| **`0xFF01_0000`** | `8 KiB` (`0xFF01_0000..1FFF`) | **Core-Local Shared SRAM** | SRAM compartilhada entre as **5 CPUs RASC do mesmo core** (layout livre para sincronização intra-core) |
| **`0xFF02_1000`** | `4 KiB` | **SDMA** (*System DMA*) | Canais de DMA de sistema, descritores, registrador de falha (`+0xF00`) e config (`+0xF04`) |
| **`0xFF02_2000`** | `4 KiB` | **VDMA** (*Video DMA*) | Controlador de DMA de vídeo (planar, AFBC, rotação, downscale) |
| **`0xFF02_3000`** | `4 KiB` | **RASC Control** | Reset e vetores de boot de RASC1..4 e doorbells inter-RASC |
| **`0xFF02_4000`** | `4 KiB` | **Core MMU Fault Controller** | Controle MMU (`+0x0C`) e 8 bancos de falha/endereço (`+0x10/+0x14`, stride `0x20`) para 8 bus masters |
| **`0xFF02_5000`** | `4 KiB` | **Watchdog / Core Timer** | Comparador de watchdog (`+0x00`), contador livre (`+0x10`) e limpeza de IRQ (`+0x14`) |
| **`0xFF02_6000`–`0xFF02_A000`** | `5 × 4 KiB` | **5 × Controladores de IRQ por RASC** | `0xFF02_6000 + rasc_id * 0x1000` (`RASC0..4`): `IRQ_STATUS`, `IRQ_ENABLE`, `IRQ_MASK2` |
| **`0xFF02_B000`** | `4 KiB` | **SYSCTRL / Core ID / Host IRQ / Spinlock** | `CORE_ID`, `CORE_CLK`, IRQ para/do host, switch-out, hardware spinlock, IRQ inter-core, `HARDWARE_ID`, revisão interna e capacidades DSL |
| **`0xFF04_0000`** | `256 KiB` stride | **`BITSTREAM`** | Front-end de bitstream, entropia (CABAC/CAVLC, decodificador booleano VP8/VP9) e predição de vetores de movimento |
| **`0xFF08_0000`** | `256 KiB` stride | **`REF`** | Subsistema de referência e cache AFBC |
| **`0xFF0C_0000`** | `256 KiB` stride | **`ME`** *(encode apenas)* | Estimação de movimento (*Motion Estimation*) |
| **`0xFF10_0000`** | `256 KiB` stride | **`MC`** | Compensação de movimento (*Motion Compensation*) |
| **`0xFF14_0000`** | `256 KiB` stride | **`TRF`** | Transformada direta/inversa e quantização |
| **`0xFF18_0000`** | `256 KiB` stride | **`DBL`** | Filtro de deblocking e SAO |
| **`0xFF2F_8000`** | `4 KiB` | **Bloco Auxiliar de Sistema** | Controle/status auxiliar de sistema (`+0x00..+0x28`, 32 bits) |

> **Padrão arquitetural (`[D]`):** Os 6 aceleradores de vídeo estão mapeados com **espaçamento fixo de `0x0004_0000` (256 KiB)**: `BITSTREAM (0xFF04_0000)`, `REF (0xFF08_0000)`, `ME (0xFF0C_0000)`, `MC (0xFF10_0000)`, `TRF (0xFF14_0000)` e `DBL (0xFF18_0000)`.

---

## 2. Bloco `SYSCTRL` / Host IRQ / Spinlock (`0xFF02_B000`) (`[O][D]`)

| Endereço | Offset | Largura | Acesso | Proveniência | Função de Hardware |
|---|---:|---:|---|---|---|
| `0xFF02_B000` | `+0x00` | 32b | R | `[D]` | **`CORE_ID`:** Índice do core físico atual (`0..1` no K1, `0..3` no K3) |
| `0xFF02_B004` | `+0x04` | 32b | R | `[D]` | **`CORE_CLK`:** Contador monotônico de relógio do core (usado em dumps e timestamps) |
| `0xFF02_B020` / `+0x24` | `+0x20`/`+0x24` | 32b | R | `[D]` | Registradores auxiliares de status/contadores de barramento |
| **`0xFF02_B028`** | **`+0x28`** | **32b** | **W** | **`[O]`** | **Trigger IRQ → Host:** Escrever **`1`** (após `CR14 = 0`) seta `IRQVE[lsid]` e dispara IRQ 74 para o Linux |
| **`0xFF02_B02C`** | **`+0x2C`** | **32b** | **W** | **`[D]`** | **Clear `IRQHOST`:** Escrever **`0`** limpa a interrupção Host → Firmware (bit 18 de `IRQ_STATUS`) |
| **`0xFF02_B030`** | **`+0x30`** | **32b** | **W** | **`[D]`** | **Core Switch-Out / Release:** Escrever **`1`** encerra a execução do core e libera seu nibble em `CORELSID` |
| `0xFF02_B034` | `+0x34` | 32b | W | `[D]` | Configuração SYSCTRL do core (recebe **`0x5F`** na inicialização) |
| **`0xFF02_B038`** | **`+0x38`** | **32b** | **R/W** | **`[D]`** | **Spinlock Atômico de Hardware:** Escrita `((core_id + 1) << 4) \| rasc_id` só prevalece se atual `== 0`; fazer readback até igualar; escrever **`0`** libera |
| **`0xFF02_B040 + c*4`** | `+0x40..+0x4C` | 32b | W | `[D]` | **Doorbell Inter-Core (core `c`):** Escrever `1` gera IRQ bit 23 (`0x800000`) no core `c`; escrever `0` limpa |
| **`0xFF02_B068`** | **`+0x68`** | **32b** | **R** | **`[D]`** | **`HARDWARE_ID` interno:** `0x56635002` no K1 e K3 (espelho interno de `0xC050_0000`) |
| **`0xFF02_B06C`** | **`+0x6C`** | **32b** | **R** | **`[D]`** | **Revisão interna do silício:** identificador interno de revisão de hardware |
| `0xFF02_B074` | `+0x74` | 32b | R | `[D]` | **Capabilities:** bit 29 (`0x2000_0000`) indica capacidade de modo DSL estendido |
| `0xFF02_B07C` | `+0x7C` | 32b | R | `[D]` | **DSL Mode:** bit 0 (`0x1`) indica precisão DSL (`0` = padrão/low, `1` = high) |

---

## 3. Bloco de Controle RASC (`0xFF02_3000`), MMU (`0xFF02_4000`), Watchdog (`0xFF02_5000`) e IRQs (`0xFF02_6000`–`0xFF02_A000`) (`[D]`)

| Bloco / Endereço | Largura | Acesso | Função |
|---|---:|---|---|
| `0xFF02_3004` / `08` / `0C` / `10` | 32b | W | Reset de RASC1 / RASC2 / RASC3 / RASC4 (`1` = assert, `0` = release) |
| `0xFF02_3024` / `28` / `2C` / `30` | 32b | W | Endereço de entrada (boot PC) de RASC1 / RASC2 / RASC3 / RASC4 |
| `0xFF02_3040` / `44` / `48` / `4C` / `50` | 32b | W | Sinal / doorbell inter-RASC canal A (`RASC0..4`) |
| `0xFF02_3060` / `64` / `68` / `6C` / `70` | 32b | W | Doorbell / ack inter-RASC canal B (`RASC0..4`, aciona/limpa bit 1 de `IRQ_STATUS`) |
| `0xFF02_400C` | 32b | W | Controle MMU interno do core (inicializado com **`0x0F`**) |
| `0xFF02_4010 + m*0x20` (`m=0..7`) | 32b | R/W | **Status de falha MMU do master `m`** (`0..4`=RASC0..4, `5`=SDMA, `6`=VDMA, `7`=REF; escrever `0` limpa) |
| `0xFF02_4014 + m*0x20` (`m=0..7`) | 32b | R | **Endereço virtual faltoso da MMU do master `m`** (VA de 32 bits que gerou a falha) |
| `0xFF02_5000` | 32b | W | **Watchdog Compare:** deadline = `[0xFF02_5010] + timeout` |
| `0xFF02_5010` | 32b | R | **Watchdog Free-Running Timer** |
| `0xFF02_5014` | 32b | W | **Watchdog IRQ Clear** (escrever `1`) |
| `0xFF02_6000 + r*0x1000` (`r=0..4`) | 32b | R | `IRQ_STATUS` da RASC `r` (b1=inter-RASC, b3=MMU, b4=Watchdog, b17=evento interno, b18=`IRQHOST`, b23=inter-core) |
| `0xFF02_6004 + r*0x1000` (`r=0..4`) | 32b | R/W | `IRQ_ENABLE` da RASC `r` |
| `0xFF02_6008 + r*0x1000` (`r=0..4`) | 32b | R/W | `IRQ_MASK2` / modo de interrupção da RASC `r` |

---

## 4. Motores DMA: `SDMA` (`0xFF02_1000`) e `VDMA` (`0xFF02_2000`) (`[D]`)

| Bloco | Endereço | Largura | Acesso | Nome / Papel Observado |
|---|---|---:|---|---|
| **SDMA** | `0xFF02_1000 + ch*4` | 32b | R/W | Status dos canais SDMA (40 pares = 80 canais `0..79`; bits `[7:5]` indicam atividade) |
| **SDMA** | `0xFF02_1600 + dev*4` | 32b | R | Pendências por dispositivo SDMA (`0..55`) |
| **SDMA** | `0xFF02_1804 + ch*0x10` | 32b | R/W | Comprimento (`srclen`/`dstlen`) do descritor do canal `ch` |
| **SDMA** | `0xFF02_180C + ch*0x10` | 32b | R/W | Controle/STP do canal `ch` (`stpen` = bit 26, `stpid` = bits `[23:16]`, bit 29 = flag de comprimento válido) |
| **SDMA** | **`0xFF02_1F00`** | 32b | R | **Falha SDMA** — registrador de falha do controlador SDMA (valor `!= 0` indica erro de canal) |
| **SDMA** | **`0xFF02_1F04`** | 32b | W | Configuração global do SDMA (inicializado com **`0xB6`**) |
| **SDMA** | `0xFF00_1C00` (`32 × 4 B`) | 32b | R/W | **Janela STP do SDMA** (32 entradas de 32 bits na Session SRAM para sincronização *Stream/Stripe Point*) |
| **VDMA** | `0xFF02_2000`–`0xFF02_229F` | 32b | R/W | Registradores do controlador **VDMA** (`0xFF02_227C..0xFF02_229C` = janela de status/diagnóstico VDMA; offsets detalhados de programação de stripes pendentes — lacuna G1) |

---

## 5. Blocos Aceleradores de Vídeo (`0xFF04_0000` – `0xFF18_0000`) (`[D]`)

Os 6 blocos aceleradores de vídeo ocupam janelas de 256 KiB em `0xFF04_0000`–`0xFF18_0000`. A tabela abaixo lista apenas as bases MMIO e os registradores de controle/status já caracterizados; a especificação completa bit a bit para disparo de quadros/stripes por codec permanece em aberto na lacuna **G1** ([observed-sequences-and-gaps.md](observed-sequences-and-gaps.md)):

| Subsistema | Base MMIO | Registradores de Controle / Status Mapeados (`Offset` → Função de Hardware) |
|---|---|---|
| **`BITSTREAM`** | **`0xFF04_0000`** | `+0x130`: bits disponíveis no buffer de entrada · `+0x208`: controle de decodificação/entropia · `+0x420`: status do sub-bloco de predição de vetores de movimento · `+0x468`: status do sub-bloco CABAC (H.264/HEVC) · `+0x514`: status do decodificador booleano VP8/VP9 · `+0x628`: status do motor aritmético auxiliar |
| **`REF`** | **`0xFF08_0000`** | `+0x00`: status do subsistema de referência · `+0x08`: configuração · `+0x44`: máscara de parada · `+0x48`: referências ativas |
| **`ME`** | **`0xFF0C_0000`** | Bloco de estimação de movimento (utilizado em **encode**; offsets internos pendentes em **G1**) |
| **`MC`** | **`0xFF10_0000`** | `+0x20`: disparo (*start*) · `+0x28`: status |
| **`TRF`** | **`0xFF14_0000`** | `+0x08`: status · `+0x64`: status de diagnóstico |
| **`DBL`** | **`0xFF18_0000`** | `+0x00`: controle principal · `+0x08`: status principal · `+0x50`: controle secundário · `+0x54`, `+0x58`, `+0x5C`: registradores adicionais de status |

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
