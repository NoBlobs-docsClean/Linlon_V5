<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Ambiente de Execução — 5 CPUs RASC por Core

Este documento especifica o ambiente de execução embarcado dentro de cada core da VPU Linlon V5: a organização das 5 CPUs RASC (Master + 4 Workers), o catálogo completo de registradores de controle (`CR0`–`CR255`), o gerenciamento de instâncias via `CR4`, os contadores e identificação via `CR16`/`CR18`, e o tratamento de exceções/pilhas.

> Para a sequência passo a passo de boot de hardware, consulte [boot-core-init.md](boot-core-init.md). Para o controlador de interrupções, doorbells e spinlock de hardware, consulte [rasc-irq-and-sync.md](rasc-irq-and-sync.md).

---

## 1. Modelo Multi-RASC por Core (`[P][D]`)

Cada core físico da VPU contém **5 processadores RASC independentes** compartilhando a Core-Local SRAM (`0xFF01_0000`, 8 KiB) e acessando a Session SRAM (`0xFF00_0000`, compartilhada entre todos os cores do LSID):

| CPU RASC | `RASC_ID` (`CR18`) | Controlador IRQ | Papel Arquitetural |
|---|---:|---|---|
| **RASC0 (Master)** | `0` | `0xFF02_6000` | Única CPU que inicia automaticamente no reset de hardware em `core_base + 0x1000`; programa vetores/resets das workers em `0xFF02_3000` e inicia o core |
| **RASC1 (Worker 1)** | `1` | `0xFF02_7000` | Processador auxiliar simétrico (papel definido pelo firmware; liberado via `0xFF02_3004`/`0xFF02_3024`) |
| **RASC2 (Worker 2)** | `2` | `0xFF02_8000` | Processador auxiliar simétrico (papel definido pelo firmware; liberado via `0xFF02_3008`/`0xFF02_3028`) |
| **RASC3 (Worker 3)** | `3` | `0xFF02_9000` | Processador auxiliar simétrico (papel definido pelo firmware; liberado via `0xFF02_300C`/`0xFF02_302C`) |
| **RASC4 (Worker 4)** | `4` | `0xFF02_A000` | Processador auxiliar simétrico (papel definido pelo firmware; liberado via `0xFF02_3010`/`0xFF02_3030`) |

- **Regra de flexibilidade para o time limpo (`[O]`):** Exceto pelo fato de apenas a **RASC0** arrancar automaticamente no reset e de cada CPU possuir seu controlador de IRQ dedicado (`0xFF02_6000 + rasc_id * 0x1000`), as 4 workers (`RASC1..4`) são processadores auxiliares simétricos cuja divisão de responsabilidades (protocolo MVE, `IRQHOST`, watchdog, parsing ou pipeline de vídeo) é livremente escolhida pelo software de firmware. Em M1/M2, a RASC0 sozinha já consegue operar todo o protocolo MVE 3.5.

---

## 2. Catálogo de Registradores de Controle (`CR0` – `CR255`) (`[O][D]`)

As instruções `mtcr rs, imm` e `mfcr rd, imm` endereçam um espaço de **256 registradores de controle** (índice de **8 bits**, `0x00`–`0xFF`, comprovado em hardware `[O]`). O espaço de CRs divide-se entre **CRs de CPU/sistema (`0x00`–`0x1F`)** e **bancos estendidos de coprocessadores/aceleradores (`0x20`–`0xFF`)**:

### 2.1 Registradores de Controle da CPU RASC (`CR 0x00` – `0x1F`)

| CR (Dec / Hex) | Acesso | Proveniência | Função Confirmada no Hardware |
|---|---:|---|---|
| **`CR0`** (`0x00`) | W | `[D]` | **VBAR (*Vector Base Address Register*):** Ponteiro para tabela de **4 vetores de 32 bits** (`[IRQ, Abort1, Abort2, Abort3]`) |
| **`CR1`** (`0x01`) | R | `[D]` | **EPC (*Exception PC*):** Endereço da instrução que gerou a exceção/abort |
| **`CR2`** (`0x02`) | R | `[D]` | **FAR (*Fault Address Register*):** Endereço virtual faltoso em exceções de memória |
| **`CR3`** (`0x03`) | W | `[D]` | **Configuração Base da CPU:** Inicializado com **`0x0000_01A7`** no boot de cada RASC |
| **`CR4`** (`0x04`) | R/W | `[D]` | **Base da Instância de Firmware (`slot << 24`):** Define qual janela de 16 MiB (`0x0000_0000`, `0x0100_0000`, `0x0200_0000`, `0x0300_0000`) mapeia esta instância de core |
| **`CR6`** (`0x06`) | W | `[D]` | **Controle de Modo / Energia da CPU:** Recebe **`0x0000_0200`** no modo ativo de boot; recebe **`0x0000_0001`** antes de `wait` em repouso final |
| **`CR11`** (`0x0B`) | W | `[D]` | **Controle de Estado/Reset Auxiliar:** Zerado (`0`) após init e antes de switch-out |
| `CR12` / `CR13` (`0x0C`/`0x0D`) | W | `[D]` | Registradores auxiliares de controle da CPU (uso esporádico; detalhes em aberto) |
| **`CR14`** (`0x0E`) | W | **`[O][D]`** | **Barreira de Memória / Write-Buffer Drain:** Escrever **`0`** garante visibilidade global de escritas em DRAM/SRAM antes de liberar spinlocks (`0xFF02_B038`), acionar doorbells ou disparar IRQ ao host (`0xFF02_B028`). **Validado em silício no K1.** |
| **`CR15`** (`0x0F`) | W | `[D]` | **Manutenção de Cache (linhas de 128 B):** Escrever endereço alinhado a `0x80` executa operação de pré-carga/invalidação na linha correspondente |
| **`CR16`** (`0x10`) | R | `[D]` | **Cycle Counter:** Contador livre de ciclos da CPU RASC (usado para medição de latência e profiling) |
| **`CR18`** (`0x12`) | R | `[D]` | **`RASC_ID`:** Identificador imutável da CPU RASC dentro do core (**`0`, `1`, `2`, `3` ou `4`**) |

### 2.2 Bancos Estendidos de Coprocessadores / Aceleradores (`CR 0x20` – `0xFF`) (`[O][D]`)

Além dos registradores de CPU (`0x00`–`0x1F`), as instruções `mtcr`/`mfcr` acessam bancos de registradores de coprocessadores/aceleradores internos acoplados ao pipeline das CPUs RASC (sem documentação pública; a semântica detalhada por registrador permanece em aberto na lacuna **G2** em [observed-sequences-and-gaps.md](observed-sequences-and-gaps.md)):

- **Achado empírico em silício (`[O]` — K1):** Testes diretos na CPU RASC0 do SpacemiT K1 comprovaram que escritas em **`CR 0x90`, `CR 0x93` e `CR 0x94`** possuem efeito colateral de cache/estado sobre a memória quando dados críticos residem em RAM não-sincronizada, e que `CR 0x94` e `CR 4` são decodificados como registradores distintos pelo hardware.
- **Aviso crítico ao time limpo (`[O]`):** Nunca mascare o número do CR para 4 bits (`& 0xF`), pois todos os 8 bits são decodificados pelo silício. Evite tocar em CRs estendidos (`>= 0x20`) não caracterizados durante o bring-up escalar; para os Marcos **M1–M3**, apenas os CRs de CPU (`CR0`, `CR1`, `CR2`, `CR3`, `CR4`, `CR6`, `CR11`, `CR14`, `CR15`, `CR16`, `CR18`) são necessários.

---

## 3. Status Register (`SR`), `clri`/`seti` e Tratamento de Exceções (`[P][D]`)

### 3.1 Contrato de Hardware — Instruções de Estado e Registradores de Exceção

- `mfsr rd` / `mtsr rs`: leitura e escrita do *Status Register* (`SR`).
- `clri rN`: **salva** estado anterior de interrupção em `rN` e **desabilita** interrupções.
- `seti rN`: **restaura** estado de interrupção a partir de `rN`.
- `wait`: coloca a CPU RASC em halt até chegada de interrupção habilitada.
- `rte`: retorna de exceção/interrupção (usado ao fim do vetor `0` de `CR0`).
- Ao entrar em qualquer um dos 4 vetores de `CR0`, o hardware expõe o endereço faltoso em **`CR2` (`FAR`)**, o endereço da instrução em **`CR1` (`PC`)** e o estado em **`SR` (`mfsr`)**. A instrução `push r0-r14` grava `r0` no menor endereço até `r14` (`lr`) no maior endereço.

### 3.2 Exemplo Ilustrativo de Frame de Exceção na Pilha (Convenção de Software)

O layout exato salvo na pilha pelo stub de IRQ/exceção é escolha livre da implementação clean-room. Como padrão ilustrativo compatível com `push r0-r14`:
1. `push r0-r14` (15 words: `r0` no menor endereço, `r14` no maior).
2. Ler `CR2` (`FAR`), `CR1` (`PC`) e `SR` (`mfsr`) em registradores temporários (ex.: `r12..r14`) e empilhá-los (`push r12-r14`, 3 words).
3. Passar `sp` em `r0` ao tratador em C:

| Offset a partir de `sp` | Tamanho | Campo (ilustrativo) |
|---:|---:|---|
| `+0x00` | 4 B | `FAR` (`CR2`) |
| `+0x04` | 4 B | `PC` (`CR1`) |
| `+0x08` | 4 B | `SR` (`mfsr`) |
| `+0x0C` .. `+0x44` | 60 B | `R[0]` .. `R[14]` em ordem crescente |

---

## 4. Regras de Pilha e Isolamento Multi-Instância (`[O][D]`)

1. Cada uma das 5 CPUs RASC **deve** possuir uma pilha exclusiva não sobreposta dentro do BSS privado da instância de core (páginas com bit `1` em `bss_bitmap` fora de `master_rw`).
2. Toda inicialização de `sp` **deve** usar `lpc sp, stack_top_rascN` (endereçamento PC-relativo), para funcionar de forma idêntica nas instâncias `0x0000_0000`, `0x0100_0000`, `0x0200_0000` e `0x0300_0000`.
3. A página `core_base + 0x0000` (`0x0000..0x0FFF`) é não-mapeada (página-guarda); qualquer página com bit `0` no `bss_bitmap` também gera `MMU ABORT` em caso de *stack overflow*.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
