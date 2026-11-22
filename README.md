<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Especificação Clean-Room — Firmware da VPU SpacemiT K1 / K3 (Arm China Linlon V5)

Esta pasta contém a especificação comportamental, de arquitetura de hardware, de ISA e de contrato ABI para uma **reimplementação em sala limpa (*clean-room*)** do firmware da VPU **Arm China Linlon V5** (`HARDWARE_ID = 0x56635002`, família de driver `MVE_v52_v76`) integrada aos SoCs **SpacemiT Key Stone K1** (2 cores a 819,2 MHz) e **SpacemiT K3** (4 cores a 1 GHz).

O alvo primário desta especificação é o contrato aberto **MVE 3.5** implementado pelo driver Linux GPL-2.0 `mvx` (`compatible = "arm china,linlon-v5"`, diretório de firmware `linlon-v52_v76-80-2/`), executando sobre as **5 CPUs RASC de 32 bits por core** da VPU.

---

## Regras de Leitura e Proveniência

Antes de iniciar qualquer implementação, leia [clean-room-process.md](clean-room-process.md).

Cada afirmação técnica nesta especificação segue três níveis de proveniência:

- **`[P]` Público (*Public*)** — Contrato normativo derivado diretamente de fontes abertas: headers GPL-2.0 do driver Linux `mvx` (`mve_protocol_def.h`, `mvx_hwreg.h`, `mvx_mmu.h`, `mvx_lsid.h`, `mvx_firmware_cache.h`), Device Trees do SpacemiT K1/K3, datasheets/manuais públicos SpacemiT, apresentações técnicas públicas da Arm China e o toolchain público `ARM-software/rasc-devkit` (GPLv2, binutils 2.18 + GCC 4.2.3).
- **`[O]` Observado em Hardware (*Observed*)** — Comportamento, valor de registrador, trace MVE ou semântica de instrução verificada empiricamente em placa real (**SpacemiT K1 BPI-F3** e/ou **SpacemiT K3 Pico ITX**), incluindo execução de código RASC próprio compilado com `rasc-elf-gcc` sobre o silício do K1.
- **`[D]` Especificação Derivada (*Derived*)** — Requisito comportamental, registrador MMIO interno, registrador de controle (`CR`) ou sequência de programação necessária para que o silício `0x56635002` responda corretamente ao contrato `[P]` e reproduza o comportamento `[O]`, descrita exclusivamente em termos de hardware e ABI pública.

---

## Mapa da Documentação

### 1. Governança, Arquitetura e Plataforma

| # | Documento | Conteúdo |
|---:|---|---|
| 01 | [clean-room-process.md](clean-room-process.md) | Firewall de sala limpa, o que entra e o que é proibido, hierarquia de fontes, protocolo de clarificação e marcos de aceitação (M1–M6) |
| 02 | [overview.md](overview.md) | Arquitetura da VPU Linlon V5 (`0x56635002`), comparação K1 (2 cores) × K3 (4 cores), modelo de 5 CPUs RASC por core e fronteira Host × Firmware |
| 03 | [open-vs-closed.md](open-vs-closed.md) | Fontes abertas (`mvx` GPL-2.0, `rasc-devkit` GPLv2) × blobs proprietários, MVE v2 vs MVE 3.5 e continuidade K1 × K3 |
| 04 | [soc-integration.md](soc-integration.md) | Integração nos SoCs K1 e K3: Device Tree, janela MMIO `0xC050_0000`, IRQ 74, clock/reset/energia, DVFS, reset de hardware e tradução DMA (`phys_cpu2vpu`/`phys_vpu2cpu`) |

### 2. Registradores Host, Escalonador e MMU

| # | Documento | Conteúdo |
|---:|---|---|
| 05 | [host-registers.md](host-registers.md) | Mapa completo dos registradores APB em `0xC050_0000`: globais (`0x00`–`0x50`), por LSID (`0x0200 + 0x40 * lsid`), bitfields, fuses e snapshots em repouso no K1/K3 |
| 06 | [lsid-and-scheduler.md](lsid-and-scheduler.md) | Contextos LSID (4 LSIDs), escalabilidade transparente de 2/4 cores, Map/Unmap/Terminate/PCB, `JOBQUEUE` (4 slots), IRQ Host↔VPU e `SWITCH` |
| 07 | [mmu-and-memory-model.md](mmu-and-memory-model.md) | MMU de 2 níveis (VA 32b → PA 40b, páginas de 4 KiB, PTE de 32 bits `AT`/`PA`/`AP`, `FLUSH_ALL`), mapa completo de VA MVE 3.x e coerência |

### 3. Contêiner de Firmware, CPU RASC e Toolchain

| # | Documento | Conteúdo |
|---:|---|---|
| 08 | [firmware-container.md](firmware-container.md) | Contêiner `.fwb`: `struct mvx_fw_header` (180 B), algoritmo de carga (`VA 0x1000`), `text_length` incluindo header, `bss_bitmap` vs `master_rw`, trampolim e nomes de arquivo |
| 09 | [rasc-isa.md](rasc-isa.md) | Referência completa da ISA RASC (1–6 B + sufixos 1–4 B, `r0`–`r15`/`v0`–`v15`, semântica **BCC3** por contagem de instruções e correções validadas em silício) |
| 10 | [rasc-execution-environment.md](rasc-execution-environment.md) | Ambiente das 5 CPUs RASC: `CR0` (VBAR), `CR1` (PC), `CR2` (FAR), `CR4` (base de instância), `CR14` (barreira), `CR15` (cache), `CR16` (ciclos), `CR18` (`RASC_ID`), bancos `CR 0x20..0xFF` e tratamento de exceções |
| 11 | [rasc-toolchain-and-abi.md](rasc-toolchain-and-abi.md) | Compilação freestanding com `ARM-software/rasc-devkit` (GCC 4.2.3 + binutils 2.18 `rasc-elf`), ABI medida (`r0`–`r4`, `r0:r1`, `r5`–`r13`), linker script e armadilhas |

### 4. Programação de Hardware Interno do Core (Boot, IRQs, Recuperação e Subsistemas)

| # | Documento | Conteúdo |
|---:|---|---|
| 12 | [boot-core-init.md](boot-core-init.md) | Sequência passo a passo de boot do core: `CR3=0x1A7`, vetores/resets RASC1..4 (`0xFF023000`), spinlock `0xFF02B038`, `CR4`, `CR15`, defaults (`0xFF02400C`, `0xFF02B034`, `0xFF021F04`), validação `0xFF02B068..7C` e switch-out `0xFF02B030` |
| 13 | [rasc-irq-and-sync.md](rasc-irq-and-sync.md) | Tabela de 4 vetores em `CR0`, `clri`/`seti` aninháveis, 5 controladores de IRQ (`0xFF026000..0xFF02A000`), bits `IRQHOST` (b18 + clear `0xFF02B02C`), `MMU` (b3), `WATCHDOG` (b4), inter-core (b23), doorbells e spinlock atômico |
| 14 | [hardware-recovery.md](hardware-recovery.md) | Watchdog (`0xFF025000`), `MMU ABORT` nos 8 bus masters (`0xFF024010+`, stride `0x20`), `RASC ABORT`, falha de SDMA (`0xFF021F00`), `DUMP`, erros MVE e `TERMINATE` |
| 15 | [internal-mmio-map.md](internal-mmio-map.md) | Mapa consolidado de `0xFF00_0000`–`0xFF2F_8FFF`: Session SRAM, Core-Local SRAM, `SDMA`, `VDMA`, `RASC_CTRL`, `MMU`, `WATCHDOG`, 5×IRQ, `SYSCTRL` e os 6 blocos de vídeo (`0xFF04_0000`..`0xFF18_0000`, stride 256 KiB) |
| 16 | [video-subsystems.md](video-subsystems.md) | Subsistemas fixos (`BITSTREAM`, `REF`, `ME`, `MC`, `TRF`, `DBL`, `SDMA`, `VDMA`, `SYSCTRL`, `MMU`), fluxos de decode/encode, `IntBuf`, DPB e coordenação multi-core |

### 5. Protocolo Host ↔ Firmware (MVE 3.5)

| # | Documento | Conteúdo |
|---:|---|---|
| 17 | [mve-queues-and-rpc.md](mve-queues-and-rpc.md) | 6 filas circulares de 4 KiB (`0x1007_9000`–`0x1007_E000`, 1020 words), empacotamento word-aligned, canal RPC (`0x1007_F000`) e RAM de print circular (`0x1010_0000`, magic `0x11223356`) |
| 18 | [mve-messages-and-states.md](mve-messages-and-states.md) | Catálogo completo de requisições (`1001`–`1013`), respostas (`2001`–`2019`), códigos de erro (`1`–`9`), eventos (`PROCESSED`, `REF_FRAME`, `TRACE_BUFFERS`) e máquina de estados (`STOPPED`/`RUNNING`) |
| 19 | [mve-options.md](mve-options.md) | Referência completa de `MVE_REQUEST_CODE_SET_OPTION` (`1010`): todos os índices (`1`–`69`), payloads estruturados e bitmasks |
| 20 | [mve-buffers-and-formats.md](mve-buffers-and-formats.md) | Descritores de buffer (`3001`–`3004`): `mve_buffer_frame` (Planar e AFBC, bitfield `format`), `mve_buffer_bitstream`, `mve_buffer_param` (19 tipos), `mve_buffer_general` e validações |

### 6. Codecs, Validação em Placa e Referência

| # | Documento | Conteúdo |
|---:|---|---|
| 21 | [codecs-and-capabilities.md](codecs-and-capabilities.md) | Matriz dos 12 codecs de decode e 5 de encode, arquivos `.fwb` (incluindo H.263 → `mpeg4dec.fwb`), formatos de pixel, fuses e restrições por codec |
| 22 | [observed-sequences-and-gaps.md](observed-sequences-and-gaps.md) | Sequências reais no K1 e K3 (decode H.264 128×128, RPC, firmware próprio em RASC), armadilhas de plataforma e registro formal das lacunas abertas (**G1–G5**) |
| 23 | [glossary.md](glossary.md) | Glossário de siglas e termos técnicos da arquitetura Linlon V5, RASC, MVE 3.5, AFBC, LSID e SpacemiT K1/K3 |

---

## Status de Cobertura para Reimplementação

- **Alvo de silício confirmado:** Arm China **Linlon V5**, `HARDWARE_ID = 0x56635002` (família `0x5663`, revisão `0x50` = 80, patch `0x02`), `FUSE = 0x00000000`, `NLSID = 4`.
  - **SpacemiT K1:** `NCORES = 2`, `CONFIG = 0xFFFF6402`, clock padrão `819,2 MHz`.
  - **SpacemiT K3:** `NCORES = 4`, `CONFIG = 0xFFFF7404`, clock padrão `1 GHz`.
- **Plano de Controle (Marcos M1, M2 e M3) — COBERTURA COMPLETA:**
  1. Contêiner `.fwb`, carga na MMU, trampolim, pilha PC-relativa, ISA escalar RASC e ABI GCC 4.2.3 validados em silício.
  2. Sequência completa de boot do core ([boot-core-init.md](boot-core-init.md)): RASC0 + RASC1..4 (`0xFF023000`), spinlock de hardware (`0xFF02B038`), `CR4` base de instância, `CR15` cache, `CR0` VBAR (4 vetores), unit defaults (`0xFF02400C`, `0xFF02B034`, `0xFF021F04`), validação `HARDWARE_ID`/DSL e switch-out (`0xFF02B030`).
  3. Controladores de interrupção por RASC (`0xFF026000..0xFF02A000`), `IRQHOST` (bit 18 + clear `0xFF02B02C`), IRQ FW→Host (`CR14=0` + `0xFF02B028=1`), Watchdog (`0xFF025000`) e `MMU ABORT` nos 8 bus masters (`0xFF024010+`).
  4. Protocolo completo MVE 3.5 (filas circulares, RPC, mensagens, opções, buffers, eventos e `ram_print`).
- **Plano de Dados de Vídeo (Marcos M4–M6) — BASE MAPEADA, SEQUÊNCIAS POR CODEC EM ABERTO:**
  - Todas as bases MMIO dos 6 aceleradores de vídeo (`0xFF04_0000`–`0xFF18_0000`, stride 256 KiB) e DMAs (`SDMA` `0xFF02_1000`, `VDMA` `0xFF02_2000`) + registradores de status/controle principais estão mapeados.
  - A programação detalhada bit a bit para disparar quadros/stripes em cada acelerador (lacunas **G1–G2** em [observed-sequences-and-gaps.md](observed-sequences-and-gaps.md)) será fechada nas rodadas dedicadas ao plano de dados (iniciando por JPEG em M4 e H.264 em M5).

---

## Licença

Esta especificação é distribuída sob a licença **Creative Commons Attribution-ShareAlike 4.0 International (`CC-BY-SA-4.0`)**. Consulte o arquivo [LICENSE](LICENSE) para os termos completos.

*(Nota: definições de ABI pública referenciadas do driver Linux `mvx` e da tabela de ISA pública do `ARM-software/rasc-devkit` permanecem sob suas licenças originais `GPL-2.0-only` / `GPLv2`.)*
