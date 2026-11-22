<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Processo Clean-Room e Firewall de Informação

Este documento define o protocolo de isolamento (*clean-room*) sob o qual esta especificação foi redigida e sob o qual o time limpo deve produzir a reimplementação do firmware da VPU **Arm China Linlon V5** (`HARDWARE_ID = 0x56635002`) para os SoCs **SpacemiT K1** e **SpacemiT K3**.

---

## 1. Separação de Papéis

O desenvolvimento segue o modelo clássico de duas equipes isoladas por um firewall documental:

1. **Equipe de Especificação (lado sujo → produção desta pasta)**
   - Inspeciona fontes públicas (driver Linux GPL-2.0 `mvx`, toolchain público `ARM-software/rasc-devkit`, datasheets e Device Trees), capturas empíricas em placas SpacemiT K1 e K3 e o comportamento observável do hardware.
   - Destila exclusivamente **fatos de hardware**, **contratos ABI públicos**, **máquinas de estados**, **especificações de ISA** e **sequências funcionais de registradores** para dentro de `VPU/docsClean/`.
2. **Equipe de Implementação (lado limpo → consumo desta pasta)**
   - Nunca teve acesso a blobs `.fwb` proprietários, desmontagens, código decompilado/levantado ou código-fonte proprietário do firmware da Arm China.
   - Lê exclusivamente esta especificação, headers públicos open-source do driver Linux (`GPL-2.0-only`), o toolchain público `ARM-software/rasc-devkit` (`GPLv2`), padrões públicos de vídeo (ITU-T H.264, H.265/HEVC, JPEG, VP8, VP9, MPEG-2/4, etc.) e os Device Trees/datasheets da SpacemiT.
   - Escreve toda a implementação do firmware (runtime RASC, `crt0`, linker script, tratador MVE 3.5, escalonador interno e controladores de blocos de vídeo) do zero, com arquitetura interna, estruturas privadas, nomes de símbolos e algoritmos próprios.

---

## 2. O Que Pode e O Que Não Pode Entrar Nesta Especificação

### Permitido

- **Contratos públicos do driver Linux GPL-2.0 (`mvx` / `vpu_k1x` / `vpu_k3`):**
  - Header de protocolo `mve_protocol_def.h`: regiões virtuais `MVE_MEM_REGION_*`, endereços de filas `MVE_COMM_*`, estruturas `mve_comm_area_*`, `mve_rpc_communication_area`, `mve_msg_header`, códigos `MVE_REQUEST_CODE_*`, `MVE_RESPONSE_CODE_*`, `MVE_BUFFER_CODE_*`, opções `MVE_SET_OPT_INDEX_*`, formatos `MVE_FORMAT_*`, descritores de buffer e eventos.
  - Registradores APB do lado host (`mvx_hwreg.h`/`mvx_hwreg.c`), campos de `JOBQUEUE`, `CORELSID`, `CTRL`, `FUSE` e sequência de LSID (`mvx_lsid.c`).
  - Formato da MMU de 2 níveis (`mvx_mmu.h`/`mvx_mmu.c`), layout do PTE de 32 bits e tradução física DMA do K1 (`phys_cpu2vpu`/`phys_vpu2cpu`).
  - Cabeçalho do contêiner `.fwb` (`struct mvx_fw_header`, 180 bytes) e regras de carga/mapeamento pelo driver (`mvx_firmware.c`, `mvx_firmware_cache.c`).
- **Especificação pública da ISA e ABI RASC (`ARM-software/rasc-devkit`):**
  - Tabela de codificação de instruções, máscaras de bits, sufixos de imediato e branch, semântica BCC3, conjunto de registradores (`r0`–`r15`, `v0`–`v15`) e convenção de chamada do `rasc-elf-gcc`.
- **Fatos empíricos de hardware medidos em silício (K1 e K3):**
  - Valores lidos dos registradores globais/LSID, frequências de clock, traces de protocolo MVE 3.5 e resultados de sondas (*probes*) próprias executadas nas CPUs RASC do K1.
- **Registradores internos de hardware sem documentação pública:**
  - Quando um registrador MMIO interno (`0xFF00_0000`–`0xFF2F_8FFF`) ou registrador de controle (`CR0`–`CR255`) é necessário para operar o silício (ex.: `0xFF02B028` para sinalizar IRQ ao host, `CR14` para sincronização, `CR15` para manutenção de cache), ele é especificado exclusivamente por seu **endereço/número**, **largura de acesso**, **direção (R/W)**, **valor/máscara exigida** e **efeito observável no hardware**, marcado explicitamente como sem documentação pública.
- **Requisitos comportamentais e máquinas de estados:**
  - Pré-condições, pós-condições, invariantes, validações de buffer/opção, transições de estado MVE e fluxos de dados entre os blocos de hardware (`BITSTREAM`, `REF`, `ME`, `MC`, `TRF`, `DBL`, `SDMA`, `VDMA`).

### Estritamente Proibido

- Trechos de código C, Assembly RASC ou pseudocódigo copiados dos blobs `.fwb` proprietários ou de qualquer ferramenta de desmontagem/decompilação/lifting.
- Endereços internos de código, tabelas, dados, BSS ou pilhas dos blobs proprietários (por exemplo, offsets internos de funções, ponteiros de tabelas internas ou layouts privados de `master_rw` do firmware original).
- Nomes de funções internas, variáveis privadas, strings de diagnóstico internas copiadas literalmente como requisito ou layouts privados de estruturas internas não expostas no contrato público `mve_protocol_def.h`.
- Estatísticas, hashes, revisões SVN, tabelas de bitmap de BSS ou artefatos de compilação extraídos dos binários proprietários da Arm China.

---

## 3. Hierarquia de Fontes e Proveniência

Em caso de dúvida ou aparente divergência entre fontes, aplique a seguinte precedência:

1. **`[P]` Fontes públicas normativas**
   - Driver Linux GPL-2.0 (`mve_protocol_def.h`, `mvx_hwreg.*`, `mvx_lsid.*`, `mvx_mmu.*`, `mvx_firmware*.*`), Device Trees (`k1-x.dtsi`, DT do K3) e `ARM-software/rasc-devkit` (`rasc-opc.c`, `rasc-dis.c`, backend `gcc/config/rasc/`).
2. **`[O]` Evidências medidas em hardware real (SpacemiT K1 BPI-F3 e K3 Pico ITX)**
   - Comportamento real do silício `0x56635002`: prevalece sobre inferências de documentação (por exemplo, as semânticas medidas no silício de `movh = <<1`, `movw = <<2`, `movn = ~imm` e CRs de 8 bits, bem como `HARDWARE_ID = 0x56635002` idêntico entre K1 e K3).
3. **`[D]` Especificação derivada de interação com o silício**
   - Sequências de inicialização, endereços MMIO internos e semântica operacional de `CR`s necessárias para que o hardware satisfaça `[P]` e `[O]`.

---

## 4. Protocolo de Clarificação (Loop de Perguntas)

Se durante a reimplementação o time limpo encontrar uma lacuna ou ambiguidade de hardware:

1. **Registre uma pergunta objetiva focada em contrato ou registrador de hardware**, por exemplo:
   - *"Quais bits do registrador interno em `0xFF0230xx` controlam o release de reset de cada CPU RASC worker (RASC1..4)?"*
   - *"Qual a sequência de programação de registradores do bloco `VDMA` para transferir um frame NV12 do buffer virtual `FRAMEBUF` para a SRAM interna?"*
   - *"Qual bit do Status Register (`SR`) habilita interrupções externas ao executar `seti`/`clri`?"*
2. **A equipe de especificação investiga o hardware/experimento e atualiza o documento correspondente em `docsClean/`** com tabelas de registradores, sequências ou regras comportamentais — nunca enviando código ou detalhes internos de implementação do firmware proprietário.
3. **Nunca presuma detalhes não verificados de outros IPs Mali-V / Linlon** (como Mali-V500/V550/V61 ou outras gerações): offsets MMIO internos, fusíveis, recursos de downscaling flexível e formatos suportados variam por família de silício.

---

## 5. Critérios de Aceitação do Firmware Clean-Room

Uma reimplementação em sala limpa é considerada funcionalmente progressiva contra o driver `mvx` (K1 e K3) conforme os marcos verificáveis abaixo:

| Marco | Requisito Verificável |
|---|---|
| **M1 — Contêiner `.fwb`, Boot RASC e Canal MVE** | Gera `.fwb` válido (header 180 B + trampolim de reset + `text_length` incluindo header + pilha PC-relativa), boota na RASC0 do K1/K3, escreve uma resposta na fila `MSG_OUTQ` (`0x1007_A000`), executa `CR14 = 0` + `0xFF02B028 = 1` e o driver Linux recebe a IRQ e decodifica a mensagem sem erro de protocolo. |
| **M2 — Handshake MVE 3.5 Completo (Plano de Controle)** | Responde `SWITCHED_IN`, processa `PING`→`PONG`, `SET_OPTION`→`SET_OPTION_CONFIRM`/`FAIL`, `JOB`→`JOB_DEQUEUED`, `GO`→`STATE_CHANGE(RUNNING)`, `STOP`→`STATE_CHANGE(STOPPED)`, `INPUT_FLUSH`/`OUTPUT_FLUSH`, `IDLE`/`IDLE_ACK`, `SWITCH`→`SWITCHED_OUT` e solicita memória via RPC (`MEM_ALLOC`/`MEM_RESIZE`/`MEM_FREE`) sem violação de máquina de estados. |
| **M3 — Multi-RASC e Multi-Core (K1 2 cores / K3 4 cores)** | Inicializa corretamente as 5 CPUs RASC por core (vetores de worker, cache via `CR15`, exceções via `FAR`/`PC`/`SR`) e coordena múltiplas instâncias de core via `master_rw` quando escalonado com `cores = 2` (K1) ou `cores = 4` (K3). |
| **M4 — Primeiro Codec Funcional (JPEG / MJPEG)** | Decodifica (e/ou codifica) frames JPEG Baseline via blocos `BITSTREAM`/`TRF`/`VDMA`, entregando saída planar (NV12/I420) bit-exata ou conforme padrão ITU-T T.81 no driver V4L2 M2M. |
| **M5 — Decode H.264 / HEVC Completo** | Executa parsing completo de bitstream (`BITSTREAM`), buffers intermediários (`IntBuf` via RPC), DPB com reordenação e referências (`REF`/AFBC), `MC` + `TRF` + `DBL` + `VDMA`, emitindo `FRAME_ALLOC_PARAM` e `SEQUENCE_PARAMETERS`, suportando troca dinâmica de resolução e saída NV12/I420/AFBC até 4K. |
| **M6 — Suite de Codecs, Encode e Robustez** | Suporte progressivo a encode (H.264/HEVC/VP8/VP9/JPEG com ME e rate control), demais codecs de decode (VP8/VP9/MPEG-2/MPEG-4/H.263/VC-1/RV/AVS/AVS2), multi-sessão concorrente em 4 LSIDs, watchdog e recuperação limpa em streams corrompidos. |

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
