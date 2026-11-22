<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Sequência de Boot e Inicialização de Hardware do Core (`boot-core-init`)

Este documento especifica a sequência de programação de registradores MMIO internos e registradores de controle (`CR`) exigida por cada core físico da VPU **Arm China Linlon V5** (`HARDWARE_ID = 0x56635002`) no SpacemiT K1 (2 cores) e K3 (4 cores) a cada boot inicial (*cold boot*) e retomada após *switch-out* (*warm boot*).

> **Proveniência:** Offsets MMIO internos, valores de inicialização e ordem de handshakes são `[D]` (especificação derivada de interação com o silício), com boot da RASC0, `CR14`, `0xFF02B028` e pilhas PC-relativas confirmados empiricamente em placa real SpacemiT K1 (`[O]`).

---

## 1. Visão Geral das Fases de Inicialização (`[O][D]`)

Quando o escalonador de hardware do VPU atribui um LSID a um ou mais cores físicos (`JOBQUEUE` + `SCHED = 1`), cada core físico inicia a execução pela **RASC0** no endereço virtual `core_base + 0x1000` (offset `0x00` do arquivo `.fwb`):

```text
1. Vetor de reset em offset 0x00 (`br _start`) salta sobre struct mvx_fw_header (180 B)
2. RASC0: configura CR3 = 0x1A7
3. RASC0: publica endereços de entrada das RASC workers em 0xFF023024..0xFF023030
          e aplica pulso de reset (1 → 0) em 0xFF023004..0xFF023010
4. RASC0: executa barreira CR14 = 0 e adquire spinlock de hardware (0xFF02B038)
5. RASC0: aloca slot de instância (0..NCORES-1), programa seu próprio CR4 = (slot << 24)
          e comunica o slot escolhido às workers (ex.: via Core-Local SRAM em 0xFF010000)
6. RASC1..4: saem de reset, configuram CR3 = 0x1A7, programam CR4 = (slot << 24)
             e opcionalmente executam manutenção/pré-carga de cache via CR15 (linhas de 128 B)
7. Cada RASC: configura CR6 = 0x200, pilha PC-relativa (`lpc sp, ...`), VBAR em CR0 (tabela de 4 vetores)
              e máscara de interrupções no seu controlador dedicado (0xFF026000 + rasc_id * 0x1000)
8. Inicialização de defaults de hardware do core (MMU 0xFF02400C, SYSCTRL 0xFF02B034, SDMA 0xFF021F04)
9. Validação de HARDWARE_ID (0xFF02B068) e capacidades DSL (0xFF02B074 / 0xFF02B07C)
10. Publicação de SWITCHED_IN em MSG_OUTQ + CR14 = 0 + IRQ host (0xFF02B028 = 1)
```

> **Nota para Marco M1/M2 (Bring-up mínimo em RASC0):** Se o firmware inicial de sala limpa executar apenas na **RASC0** (sem acordar RASC1–RASC4), os passos 3, 5 e 6 podem ser omitidos; basta executar o trampolim `br _start`, inicializar `sp` via `lpc`, configurar `CR3 = 0x1A7` e `CR0` (VBAR), e operar diretamente sobre as filas MVE (`0x1007_9000`–`0x1007_F000`) + `CR14 = 0` + `0xFF02B028 = 1` (`[O]`).

---

## 2. Fase A — Arranque da RASC0 e Liberação das RASC Workers (`[D]`)

### 2.1 Configuração Inicial da CPU e Publicação de Workers

1. **Configuração base da CPU:** Escrever **`0x0000_01A7`** em **`CR3`** (`mtcr r, 3`).
2. **Vetores de entrada das RASC workers (bloco `0xFF02_3000`):**
   - Calcular o endereço de entrada de cada worker via `lpc` (relativo ao PC).
   - Escrever os endereços de entrada nos registradores de vetor de reset:

   | Registrador | Largura | Função |
   |---|---:|---|
   | `0xFF02_3024` | 32b W | Endereço de entrada (boot) da **RASC1** |
   | `0xFF02_3028` | 32b W | Endereço de entrada (boot) da **RASC2** |
   | `0xFF02_302C` | 32b W | Endereço de entrada (boot) da **RASC3** |
   | `0xFF02_3030` | 32b W | Endereço de entrada (boot) da **RASC4** |

3. **Pulso de reset das RASC workers (bloco `0xFF02_3000`):**
   - Para cada worker que deseja ativar, escrever **`1`** e em seguida **`0`** no respectivo registrador de reset (`1` = assert reset, `0` = release/run):

   | Registrador | Largura | Função |
   |---|---:|---|
   | `0xFF02_3004` | 32b W | Reset da **RASC1** (`1` → `0`) |
   | `0xFF02_3008` | 32b W | Reset da **RASC2** (`1` → `0`) |
   | `0xFF02_300C` | 32b W | Reset da **RASC3** (`1` → `0`) |
   | `0xFF02_3010` | 32b W | Reset da **RASC4** (`1` → `0`) |

   *(Observação: RASC1 e RASC2 podem ser liberadas logo no início pela RASC0; RASC3 e RASC4 podem ser liberadas em seguida pela RASC0 ou por uma worker já ativa, antes de iniciar o pipeline de vídeo.)*

4. **Barreira de memória:** Escrever **`0`** em **`CR14`** (`mtcr r, 0x0E`).

---

## 3. Fase B — Eleição de Slot de Instância e Programação de `CR4` (`[D]`)

Em sessões multi-core (`NCORES = 2` no K1, `4` no K3), cada core físico que entra em execução precisa obter um **índice de instância único** (`slot` em `0 .. ncores - 1`) para mapear seu BSS privado (`slot * 0x0100_0000`) sem colisão com outros cores:

1. **Adquirir o spinlock de hardware (`0xFF02_B038`):**
   - Ler `core_id = *(volatile uint32_t *)0xFF02B000` (`0 .. NCORES - 1`).
   - Formar qualquer token não-zero único por requisitante (regra de hardware: `0xFF02_B038` aceita qualquer valor `!= 0` quando está livre em `0`; uma convenção simples é `token = ((core_id + 1) << 4) | rasc_id`).
   - Escrever `token` em `0xFF02B038` e fazer **poll obrigatório** lendo `0xFF02B038` até ler exatamente `token` de volta (ver [rasc-irq-and-sync.md](rasc-irq-and-sync.md)).
2. **Alocar slot livre na Session SRAM (`0xFF00_0000`):**
   - Como `0xFF00_0000` é SRAM compartilhada por todos os cores do LSID, o firmware escolhe um slot livre `slot` (`0 .. ncores - 1`) e o marca como ocupado sob proteção do spinlock.
   - Se nenhum slot estiver disponível temporariamente, libera o spinlock (`CR14 = 0`, `0xFF02B038 = 0`), aguarda um curto atraso e tenta novamente.
3. **Configurar registrador de base de instância (`CR4`) — contrato de hardware:**
   - Escrever **`slot << 24`** (`0x0000_0000`, `0x0100_0000`, `0x0200_0000`, `0x0300_0000`, ...) em **`CR4`** (`mtcr r, 4`).
   - Este registrador informa à lógica de endereçamento da CPU RASC qual janela de 16 MiB de instância (`MVE_MEM_REGION_FW_INSTANCE<slot>`) pertence a este core.
4. **Propagar o slot às RASC workers deste core via Core-Local SRAM (`0xFF01_0000`):**
   - **Contrato de hardware:** A janela `0xFF01_0000`–`0xFF01_1FFF` (8 KiB) é SRAM compartilhada exclusivamente entre as 5 CPUs RASC do mesmo core; o layout interno é inteiramente definido pelo software de firmware.
   - **Convenção ilustrativa de sincronização:** A RASC0 pode publicar `(slot << 24) | 1` (bit 0 como flag de pronto, permitindo distinguir `slot == 0` de zero inicial) em uma palavra escolhida da Core-Local SRAM (ex.: `0xFF01_0000`), executar **`CR14 = 0`** para visibilidade local e então liberar o spinlock escrevendo **`0`** em `0xFF02B038`.

---

## 4. Fase C — Sequência de Arranque de Cada RASC Worker (`RASC1..4`) (`[D]`)

Cada CPU RASC worker executa os seguintes passos ao sair de reset:

1. **Configuração da CPU (contrato de hardware):** Escrever **`0x0000_01A7`** em **`CR3`**.
2. **Sincronização do slot e manutenção opcional de cache (`CR15`):**
   - **Contrato de hardware:** Cada worker precisa programar seu próprio **`CR4 = slot << 24`** com o mesmo `slot` alocado para o core, e o registrador **`CR15`** aceita endereços alinhados a **128 bytes** (`addr & ~0x7F`, passo `0x80`) para manutenção/pré-carga de linha de cache.
   - **Exemplo ilustrativo de espera:** Enquanto aguarda o sinal da RASC0 na Core-Local SRAM (ex.: `v = *(volatile uint32_t *)0xFF010000`, aguardando `v != 0` na convenção `(slot << 24) | 1`), a worker pode percorrer `[inicio, fim)` de seu código (calculado via `lpc`, alinhado a `0x80`) gravando `addr` em `CR15` e então programar **`CR4 = v & ~1`**.
3. **Configurar controle/pilha (contrato de hardware):**
   - Escrever **`0x0000_0200`** em **`CR6`**.
   - Carregar a pilha exclusiva desta worker via **`lpc sp, stack_top_rascN`**.
4. **Configurar tabela de vetores (`CR0`, contrato de hardware):**
   - Carregar via `lpc` o endereço de uma tabela alinhada de 4 ponteiros de 32 bits (`[irq_handler, abort0, abort1, abort2]`) e escrevê-lo em **`CR0`** (`mtcr r, 0`).
5. **Chamar a função principal da worker** (`bsr worker_main`). Ao retornar/encerrar, executar `wait; br .`.

*(Na RASC0, após configurar `CR4`, ela igualmente configura `CR6 = 0x200`, `lpc sp, stack_top_rasc0`, copia eventuais dados `.data` da ROM/texto para BSS na primeira carga, zera **`CR11 = 0`**, configura **`CR0`** com sua tabela de vetores e segue para a inicialização dos blocos.)*

---

## 5. Fase D — Inicialização dos Blocos do Core e Validação de Silício (`[D]`)

Antes de aceitar comandos de processamento de vídeo ou interrupções de host, o core deve programar os registradores de configuração padrão do silício `0x56635002`:

### 5.1 Valores de Inicialização de Hardware do Core

| Registrador MMIO | Largura | Valor Obrigatório | Papel de Hardware |
|---|---:|---:|---|
| **`0xFF02_400C`** | 32b W | **`0x0000_000F`** | Configuração / habilitação do bloco de controle MMU interno do core |
| **`0xFF02_B034`** | 32b W | **`0x0000_005F`** | Configuração inicial de SYSCTRL do core |
| **`0xFF02_1F04`** | 32b W | **`0x0000_00B6`** | Configuração global inicial do controlador **SDMA** |

### 5.2 Validação de Versão de Hardware e Modo DSL (`0xFF02_B068`..`0xFF02_B07C`)

O firmware pode verificar em tempo de execução o silício em que está rodando diretamente pelo bloco SYSCTRL interno (`0xFF02_B000`):

| Registrador MMIO | Largura | Acesso | Função |
|---|---:|---|---|
| **`0xFF02_B068`** | 32b | R | **`HARDWARE_ID` interno:** retorna o mesmo valor do registrador host `0xC050_0000` (`0x56635002` no K1 e K3). Para checagem de família Linlon V5, comparar `(val >> 16) == 0x5663` (ou `(val & 0xFFFF0000) == 0x56630000`), mascarando revisão e patch. |
| **`0xFF02_B06C`** | 32b | R | **Revisão interna de hardware:** identificador interno de revisão do silício (útil em dumps de diagnóstico). |
| **`0xFF02_B074`** | 32b | R | **Capacidades de hardware:** bit 29 (`0x2000_0000`) indica suporte a modo de precisão estendida de downscale flexível (DSL). |
| **`0xFF02_B07C`** | 32b | R | **Modo DSL ativo:** bit 0 (`0x0000_0001`) indica se o hardware reporta modo DSL de alta precisão (`1`) ou precisão padrão (`0`). |

### 5.3 Habilitação dos Controladores de Interrupção por RASC

Cada RASC habilita as fontes de interrupção desejadas em seu banco `0xFF02_6000 + rasc_id * 0x1000` (detalhes completos em [rasc-irq-and-sync.md](rasc-irq-and-sync.md)) e executa `seti` para habilitar interrupções na CPU.

---

## 6. Fase E — Sequência de Desligamento / Switch-Out do Core (`0xFF02_B030`) (`[D]`)

Quando o core deve sair de execução (em resposta a `MVE_REQUEST_CODE_SWITCH` após drenar o trabalho e enviar `MVE_RESPONSE_CODE_SWITCHED_OUT` no último core ativo):

1. Desabilitar interrupções locais (`clri`) e limpar o controlador de interrupção da RASC conforme apropriado.
2. Zerar `CR11 = 0` e executar barreira **`CR14 = 0`**.
3. Se for o último core da sessão a sair, certificar-se de que `MVE_RESPONSE_CODE_SWITCHED_OUT` foi gravado em `MSG_OUTQ`, seguido de `CR14 = 0` e `0xFF02B028 = 1` (IRQ ao host).
4. Liberar o slot de instância do core na Session SRAM (`0xFF00_0000`) e executar **`CR14 = 0`**.
5. **Acionar liberação do core no escalonador de hardware:**
   - Escrever **`1`** em **`*(volatile uint32_t *)0xFF02B030`**.
   - Entrar em loop infinito / parada (`br .` ou `wait; br .`).
   - O hardware retira o LSID do core correspondente em `CORELSID` (nibble volta a `0xF`), tornando o core disponível para outro LSID ou para `runtime_suspend`.

*(Observação sobre modo `wait` de RASC0 ao entrar em repouso: a RASC0 escreve **`CR6 = 1`** antes de entrar em `wait; br .` ao encerrar sua rotina de inicialização.)*

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
