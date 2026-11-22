<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Fontes Abertas × Blobs Proprietários e Compatibilidade entre Gerações (`open-vs-closed`)

Este documento esclarece qual é o contrato aberto alvo desta especificação, o que pertence ao ecossistema open-source versus blobs fechados, como os fatos de hardware se mantêm consistentes entre os SoCs alvo (**SpacemiT K1** e **SpacemiT K3**) e a diferença entre as revisões do protocolo MVE (v2 × v3.5).

---

## 1. O Que É Aberto e O Que É Fechado na Pilha VPU (`[P][O]`)

| Camada | Status de Licença | Papel na Reimplementação Clean-Room |
|---|---|---|
| **Driver Linux `mvx`** (`vpu_k1x` / `vpu_k3`) | **Open-source (`GPL-2.0-only`)** | **Contrato normativo `[P]`:** define registradores APB (`0xC050_0000`), LSID, MMU, formato `.fwb` (`struct mvx_fw_header`), protocolo MVE 3.x (`mve_protocol_def.h`) e V4L2 M2M. |
| **Toolchain `ARM-software/rasc-devkit`** | **Open-source (`GPLv2`)** | **Contrato normativo `[P]`:** define a tabela completa da ISA RASC (`rasc-opc.c`), o assembler/linker (`binutils 2.18`) e o compilador C (`GCC 4.2.3`) + ABI `rasc-elf`. |
| **Device Trees e Datasheets SpacemiT K1/K3** | **Público** | **Contrato de plataforma `[P]`:** endereços base, IRQ 74, clocks, OPPs, resets, domínios de energia e tradução DMA. |
| **Blobs `linlon-v52_v76-80-2/*.fwb`** | **Fechado / Proprietário** | **Proibido ao time limpo:** é exatamente o componente que esta especificação permite substituir do zero em sala limpa. |

---

## 2. Contrato Alvo: Protocolo MVE 3.5 (vs. Legado MVE v2) (`[P]`)

O driver Linux `mvx` suporta duas versões principais do protocolo host↔firmware (`protocol_major = 2` e `protocol_major = 3`):

| Aspecto | MVE v2 (Legado: Mali-V500/V550/V61) | **MVE 3.5 (Alvo desta Especificação: Linlon V5 K1/K3)** |
|---|---|---|
| Cabeçalho `.fwb` | `protocol_major = 2` | **`protocol_major = 3`, `protocol_minor = 5`** |
| Mapa de regiões virtuais | `fw_v2/mve_protocol_def.h` | **`fw_v3/mve_protocol_def.h`** (8 instâncias FW, filas `0x1007_9000..F000`, `ram_print` `0x1010_0000`, `PROTECTED` `0x2000_0000`, `FRAMEBUF` `0x7000_0000`) |
| `IDLE_ACK` (`1012`) | Ausente em v2 | **Ativo quando `major == 3 && minor >= 1`** |
| H.264 High 10 (`MVE_OPT_PROFILE_H264_HIGH_10`) | Não mapeado em v2 | **Suportado em v3** (`to_mve_profile_v3`) |
| Downscale arbitrário (DSL) e opções estendidas (`57..69`) | Parcial/ausente | **Presente no contrato MVE 3.5** |

**Recomendação normativa:** A reimplementação em sala limpa deve declarar **`protocol_major = 3`, `protocol_minor = 5`** no cabeçalho `.fwb` e seguir exclusivamente o contrato **MVE 3.5** documentado nesta pasta.

---

## 3. Continuidade de Hardware: SpacemiT K1 × SpacemiT K3 (`[P][O]`)

A verificação empírica entre os dois SoCs SpacemiT alvo mostra que ambos instanciam exatamente o mesmo IP de VPU Linlon V5 (`0x56635002`), diferindo apenas na contagem de cores e frequência operacional:

| Plataforma | Família `HARDWARE_ID` | `NCORES` | Protocolo MVE | ISA / ABI RASC | Implicação para o Time Limpo |
|---|---|---:|---|---|---|
| **SpacemiT K1** (BPI-F3) | `0x56635002` (`Linlon V5`) | 2 | **3.5** | RASC 32-bit | **Alvo primário** (validado ao vivo com código RASC próprio) |
| **SpacemiT K3** (Pico ITX) | `0x56635002` (`Linlon V5`) | 4 | **3.5** | **Idêntica** | **Alvo primário idêntico:** mesmo binário `.fwb` roda nos dois SoCs; topologia de 4 cores é guiada pelo host |

---

## 4. Divergências Conhecidas Entre Revisão de Silício e Driver (`[P][O]`)

1. **`HARDWARE_ID` do silício (`0x56635002`) vs. enum do driver (`0x5276`):**
   - O log de boot do kernel exibe `hw_id=00005276` (valor do enum `MVE_v52_v76`), enquanto o registrador físico `0xC050_0000` (e o registrador interno `0xFF02_B068`) retorna **`0x56635002`** (família `0x5663`, rev `0x50` = 80, patch `0x02`).
   - A checagem de família no firmware deve mascarar revisão/patch (ex.: `(id & 0xFFFF0000) == 0x56630000`) para aceitar todas as revisões da família Linlon V5 (`0x5663`).
2. **Kernel Mainline vs. Vendor:**
   - O kernel Linux mainline atual ainda não inclui o driver `mvx` nem o nó DT `linlon-v5@c0500000`; portanto, o desenvolvimento e validação em placa real utilizam o driver `mvx` (GPL-2.0, compilável como módulo `amvx.ko`) sobre kernel vendor ou portado com o nó DT documentado em [soc-integration.md](soc-integration.md).

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
