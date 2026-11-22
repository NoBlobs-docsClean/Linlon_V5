<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Contextos LSID, Escalonador de Hardware e Multi-Core

Este documento especifica o modelo de contextos de sessão (**LSID — Local Session ID**), o ciclo de vida de mapeamento/desmapeamento no hardware, a fila de jobs (`JOBQUEUE`), o fluxo de interrupções entre host e VPU e como a topologia multi-core (2 cores no K1, 4 cores no K3) é conduzida pelo escalonador de hardware.

---

## 1. Modelo de LSID e Multiplexação de Sessões (`[P]`)

A VPU Linlon V5 possui **`NLSID = 4`** bancos de registradores de sessão independentes (`LSID 0..3`). Cada LSID encapsula:
- Sua própria raiz de MMU (`MMU_CTRL`), isolando completamente o espaço virtual de 32 bits da sessão;
- Suas restrições de cores (`CTRL`: `DISALLOW` + `MAXCORES`);
- Seus canais independentes de interrupção (`LIRQVE` e `IRQHOST`);
- Seus atributos de barramento AXI (`BUSATTR_0..3`) e estado de proteção (`ALLOC`/`NPROT`).

Quando existem mais de 4 sessões V4L2 abertas simultaneamente, o escalonador do driver multiplexa sessões sobre os 4 LSIDs de hardware salvando e restaurando o **PCB (*Process Control Block*)** de cada sessão:

```c
struct mvx_lsid_pcb {
    uint32_t ctrl;
    uint32_t mmu_ctrl;
    uint32_t irqhost;
    uint32_t nprot;
};
```

---

## 2. Sequências de Hardware do Ciclo de Vida de um LSID (`[P]`)

### 2.1 Mapeamento de uma Sessão em um LSID (`mvx_lsid_map`)

Pré-condição: o LSID escolhido deve estar livre (`ALLOC == 0`).

1. Ler `ALLOC` (`+0x0C`) e confirmar que vale `MVE_ALLOC_FREE` (`0`).
2. Escrever `ALLOC`:
   - `MVE_ALLOC_PROTECTED` (`2`) se `pcb->nprot == 0` (sessão protegida), ou
   - `MVE_ALLOC_NON_PROTECTED` (`1`) caso contrário.
3. Garantir estado limpo executando `TERMINATE`:
   - Escrever `1` em `TERMINATE` (`+0x18`).
   - Fazer poll em `TERMINATE` até ler `0`.
4. Configurar topologia de cores em `CTRL` (`+0x00`) = `pcb->ctrl` (`(maxcores << 8) | disallow`).
5. Configurar a tabela de páginas L1 em `MMU_CTRL` (`+0x04`) = `pcb->mmu_ctrl`.
6. Invalidar a TLB escrevendo `0` em `FLUSH_ALL` (`+0x10`).
7. Configurar `BUSATTR_0..3` (`+0x30..+0x3C`): valores do Device Tree (se presentes) ou default do driver (`0x00, 0x00, 0x33, 0x33`).
8. Restaurar estado de interrupções:
   - `LIRQVE` (`+0x1C`) = `0`.
   - `IRQHOST` (`+0x20`) = `pcb->irqhost`.
9. Executar barreira de escrita (`wmb()`) para garantir visibilidade de todos os registradores antes de ativar o escalonador.
10. Habilitar o escalonamento escrevendo `1` em `SCHED` (`+0x14`).

### 2.2 Verificação de Ociosidade de um LSID (`mvx_lsid_idle`)

Um LSID só é considerado **ocioso** (e portanto seguro para desmapeamento) quando **ambas** as condições são verdadeiras:
1. O número do LSID **não** aparece em nenhum dos 4 slots de `JOBQUEUE` (`0x14`).
2. O número do LSID **não** aparece em nenhum dos `NCORES` nibbles de `CORELSID` (`0x10`).

### 2.3 Desmapeamento de um LSID (`mvx_lsid_unmap`)

Pré-condição obrigatória: o LSID deve estar ocioso (`mvx_lsid_idle == true`).

1. Desabilitar o escalonamento escrevendo `0` em `SCHED` (`+0x14`).
2. Salvar no PCB da sessão: `ctrl`, `mmu_ctrl`, `irqhost` e `nprot`.
3. Liberar o LSID escrevendo `MVE_ALLOC_FREE` (`0`) em `ALLOC` (`+0x0C`).

### 2.4 Término Forçado (`mvx_lsid_terminate`)

Usado no fechamento de sessão ou recuperação de travamento/watchdog:
1. Remover o LSID da `JOBQUEUE` (se presente).
2. Escrever `1` em `TERMINATE` (`+0x18`).
3. Fazer poll em `TERMINATE` até o hardware devolvê-lo como `0`.

---

## 3. Operação da `JOBQUEUE` (`[P]`)

### 3.1 Enfileiramento (`mvx_lsid_jobqueue_add`)

1. Escrever `0` em `ENABLE` (`0x04`) para pausar o escalonador global.
2. Ler `JOBQUEUE` (`0x14`).
3. Verificar se o LSID já está em algum dos 4 slots (`0..3`); se já estiver, pular direto para o passo 5.
4. Procurar o primeiro slot cujo campo `LSID` (`bits [3:0]`) seja `0xF` (`MVE_JOBQUEUE_JOB_INVALID`):
   - Preencher o slot com `(lsid & 0xF) | (((ncores - 1) & 0xF) << 4)`.
   - Escrever o valor atualizado em `JOBQUEUE`.
5. Escrever `1` em `ENABLE` (`0x04`) para retomar o escalonador global.

### 3.2 Remoção (`mvx_lsid_jobqueue_remove`)

1. Escrever `0` em `ENABLE`.
2. Ler `JOBQUEUE`, compactar os slots cujo LSID não corresponde ao removido mantendo a ordem FIFO, e preencher os slots restantes com `LSID = 0xF` e `NCORES-1 = ncores - 1`.
3. Escrever `JOBQUEUE` e reabilitar `ENABLE = 1`.

---

## 4. Fluxo de Interrupções Host ↔ VPU (`[P][O]`)

### 4.1 Host → Firmware

Quando o driver escreve novas mensagens/buffers nas filas `*_HOST` ou responde a um RPC:
1. Executa barreira de escrita (`wmb()`) e sincronização de cache DMA (`dma_sync_single_for_device`).
2. Atualiza `in_wpos` (ou `rpc_area->state = MVE_RPC_STATE_RETURN`) com nova barreira/sync.
3. Escreve **`1` em `IRQHOST`** (`LSID_BASE + 0x20`), acordando os cores alocados ao LSID.

### 4.2 Firmware → Host

Quando o firmware escreve respostas em `MSG_OUTQ`, devolve buffers em `BUF_INRQ`/`BUF_OUTRQ` ou solicita um RPC (`state = MVE_RPC_STATE_PARAM`):
1. O firmware garante a visibilidade da memória (escrevendo `0` em `CR14`, ver [internal-mmio-map.md](internal-mmio-map.md)).
2. O firmware escreve **`1` em `0xFF02B028`** no barramento MMIO interno do core (`[O]`).
3. O hardware seta o bit correspondente ao LSID em `IRQVE` (`0x18`) e dispara a IRQ `74` para a CPU RISC-V.
4. No *top half* (`irq_top`):
   - O host lê `NLSID` e `IRQVE`.
   - Para cada LSID com bit ativo em `IRQVE`, escreve **`0` em `LIRQVE`** (`LSID_BASE + 0x1C`), executa `wmb()`, marca o LSID em um bitmap pendente e agenda a workqueue (`irq_bottom`).
5. No *bottom half* (`irq_bottom`):
   - Para cada LSID marcado, o host lê e processa todas as mensagens pendentes nas filas de saída, trata pedidos RPC e drena `ram_print`.

---

## 5. Troca de Contexto (`SWITCH` / `SWITCHED_IN` / `SWITCHED_OUT`) (`[P][O]`)

Como o hardware suporta múltiplos LSIDs disputando os mesmos cores físicos, o firmware deve colaborar com o escalonador do host:

```text
  [Core Livre] ──(JOBQUEUE + SCHED=1)──► Firmware boota / retoma LSID
                                              │
                                              ▼
                                 Envia MVE_RESPONSE_CODE_SWITCHED_IN {core}
                                              │
                                              ▼
                                 Processa mensagens, buffers e frames
                                              │
                        ┌─────────────────────┴─────────────────────┐
                        │ (Sem trabalho restante)                    │ (Host envia SWITCH)
                        ▼                                            ▼
             Envia MVE_RESPONSE_CODE_IDLE                Drena/salva contexto do LSID
                        │                                            │
                        ▼                                            │
             Host responde MVE_REQUEST_CODE_IDLE_ACK                 │
             e/ou MVE_REQUEST_CODE_SWITCH                            │
                        │                                            │
                        └─────────────────────┬─────────────────────┘
                                              ▼
                       Envia MVE_RESPONSE_CODE_SWITCHED_OUT {core, reason, sub_reason}
                                              │
                                              ▼
                       Core libera LSID (CORELSID volta a 0xF) e entra em wait
```

- **Observado em hardware (`[O]`):** Ao encerrar o processamento dos buffers disponíveis de uma sessão H.264, o firmware envia `IDLE`, o host envia `SWITCH`, e o firmware responde `SWITCHED_OUT(core=0, reason=1, sub_reason=0)`, liberando o core (`CORELSID` retorna para `0x0000_00FF` no K1). Quando novo trabalho chega (ex.: `STOP`/`FLUSH` ou novos buffers), o host reinsere o LSID em `JOBQUEUE` e o firmware volta a emitir `SWITCHED_IN`.

---

## 6. Transparência de Topologia Multi-Core (2 Cores K1 × 4 Cores K3) (`[P][O]`)

O firmware **não tem acesso** ao bloco de registradores globais (`0xC050_0000`) nem lê `NCORES`. Toda a adaptação entre K1 (2 cores) e K3 (4 cores) ocorre por quatro mecanismos dirigidos pelo host:

1. **Mapeamento MMU de instâncias:** O driver mapeia o mesmo código executável (`text`) e a mesma região compartilhada (`master_rw`) em cada instância de core `0 .. ncores-1` (`0x0000_0000`, `0x0100_0000`, `0x0200_0000`, `0x0300_0000`), alocando páginas de BSS **privadas** distintas para cada instância.
2. **`CTRL` do LSID:** Define `MAXCORES` e `DISALLOW` conforme o `NCORES` do SoC.
3. **`JOBQUEUE`:** Informa `(ncores - 1)` no slot de escalonamento de hardware.
4. **Mensagem `MVE_REQUEST_CODE_JOB`:** Informa o campo `uint16_t cores` (entre `1` e `NCORES`) ao firmware. O firmware valida `cores >= 1` e coordena os cores participantes do job utilizando a região compartilhada `master_rw`. A alteração dinâmica do número de cores durante um job ativo não é suportada.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
