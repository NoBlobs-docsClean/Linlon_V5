<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Registradores do Lado Host (APB — Base `0xC050_0000`)

Este documento especifica todos os registradores MMIO expostos pela VPU Linlon V5 ao processador hospedeiro (CPU RISC-V) através do barramento APB3 a partir da base `0xC050_0000`. Todos os acessos são de **32 bits (little-endian)**, alinhados a 4 bytes.

---

## 1. Registradores Globais (`[P][O]`)

Offsets relativos à base `0xC050_0000`:

| Offset | Nome | Acesso | Função |
|---:|---|---|---|
| `0x00` | `HARDWARE_ID` | R | Identificação do IP: família, revisão e patch |
| `0x04` | `ENABLE` | R/W | Habilita (`1`) ou pausa (`0`) o escalonador global de jobs |
| `0x08` | `NCORES` | R | Número de cores físicos da VPU presentes no silício (K1 = `2`, K3 = `4`) |
| `0x0C` | `NLSID` | R | Número de contextos de sessão (LSIDs) suportados pelo hardware (`4`) |
| `0x10` | `CORELSID` | R | LSID atualmente em execução em cada core físico (4 bits por core; `0xF` = livre) |
| `0x14` | `JOBQUEUE` | R/W | Fila de hardware com 4 slots de 8 bits `{ncores-1[7:4], lsid[3:0]}` |
| `0x18` | `IRQVE` | R | Máscara global de interrupções pendentes por LSID (bit `i` = LSID `i`) |
| `0x24` | `CLKFORCE` | R/W | Controle de clock forçado (`0` após reset para habilitar clock gating) |
| `0x34` | `FUSE` | R | Máscara de fusíveis de desabilitação de recursos/codecs |
| `0x38` | `CONFIG` | R | Registrador de configuração/topologia do silício |
| `0x40` | `PROTCTRL` | R/W | Controle global de sessões protegidas |
| `0x50` | `RESET` | W/R | Escrever `1` dispara reset interno do bloco VPU |

---

## 2. Detalhamento de Campos dos Registradores Globais (`[P][O]`)

### 2.1 `HARDWARE_ID` (`0x00`)

```text
 31                16 15         8 7          0
+--------------------+------------+------------+
|     FAMILY_ID      |  REVISION  |   PATCH    |
+--------------------+------------+------------+
```

| Bits | Campo | Valores Conhecidos |
|---|---|---|
| `[31:16]` | `FAMILY_ID` | `0x5650` = Mali-V500; `0x5655` = Mali-V550; `0x5660`/`0x5661` = Mali-V61; **`0x5662`/`0x5663` = `MVE_v52_v76` (Linlon V5/V7)** |
| `[15:8]` | `REVISION` | `0x50` (80 decimal) no K1 e no K3 |
| `[7:0]` | `PATCH` | `0x02` no K1 e no K3 |

- **Valor medido em placa (K1 e K3):** `0x56635002` (`[O]`).

### 2.2 `CORELSID` (`0x10`)

Cada core físico `c` (`0 <= c < NCORES`) ocupa um nibble de 4 bits:

```text
core_lsid(c) = (CORELSID >> (4 * c)) & 0xF
```

- `0..3`: o core `c` está executando a sessão mapeada no LSID indicado.
- `0xF` (`15`): o core `c` está **livre / ocioso**.
- Em repouso: K1 (`NCORES = 2`) lê `0x0000_00FF`; K3 (`NCORES = 4`) lê `0x0000_FFFF` (`[O]`).

### 2.3 `JOBQUEUE` (`0x14`)

Contém **4 slots de 8 bits** (`nr = 0..3`, LSB para MSB):

```text
 31       24 23       16 15        8 7         0
+-----------+-----------+-----------+-----------+
|  SLOT 3   |  SLOT 2   |  SLOT 1   |  SLOT 0   |
+-----------+-----------+-----------+-----------+

Cada SLOT (8 bits):
  7       4 3       0
 +---------+---------+
 | NCORES-1|  LSID   |
 +---------+---------+
```

| Campo do Slot | Bits | Constante / Regra |
|---|---|---|
| `LSID` | `[3:0]` | `0..3` = LSID enfileirado; **`0xF`** (`MVE_JOBQUEUE_JOB_INVALID`) = slot vazio/inválido |
| `NCORES-1` | `[7:4]` | Número de cores solicitados para o job menos 1 (`0` = 1 core, `1` = 2 cores, `3` = 4 cores) |

- Em repouso (fila vazia): `0x0F0F_0F0F` (`[O]`).
- **Protocolo de modificação (`[P]`):** Antes de ler-modificar-escrever `JOBQUEUE`, o host **deve** desabilitar o escalonador (`ENABLE = 0`), atualizar `JOBQUEUE` e reabilitar (`ENABLE = 1`).

### 2.4 `IRQVE` (`0x18`)

- Bit `i` (`1 << i`, para `0 <= i < NLSID`) indica que o LSID `i` possui uma interrupção ativa endereçada ao host (disparada pelo firmware via registrador interno `0xFF02B028`).
- A limpeza da interrupção é feita no registrador por LSID `LIRQVE` (escrevendo `0`), não em `IRQVE`.

### 2.5 `FUSE` (`0x34`)

Cada bit em `1` **desabilita** o recurso correspondente no hardware:

| Bit | Máscara | Macro Pública | Efeito quando `1` |
|---:|---|---|---|
| 0 | `0x0000_0001` | `MVX_HWREG_FUSE_DISABLE_AFBC` | Desabilita formatos AFBC (8-bit e 10-bit, 4:2:0 e 4:2:2) |
| 1 | `0x0000_0002` | `MVX_HWREG_FUSE_DISABLE_REAL` | Desabilita decodificação RealVideo (`RV`) |
| 2 | `0x0000_0004` | `MVX_HWREG_FUSE_DISABLE_VPX` | Desabilita codecs VP8 e VP9 |
| 3 | `0x0000_0008` | `MVX_HWREG_FUSE_DISABLE_HEVC` | Desabilita codec HEVC (H.265) |

- **Valor medido em placa (K1 e K3):** `0x0000_0000` (nenhum recurso desabilitado) (`[O]`).

### 2.6 `CONFIG` (`0x38`) e `PROTCTRL` (`0x40`)

- Os campos individuais de bits de `CONFIG` e `PROTCTRL` não possuem macros públicas no driver Linux (`[P]` não define campos; valores medidos `[O]`):
  - `CONFIG`: `0xFFFF6402` no K1 (2 cores) e `0xFFFF7404` no K3 (4 cores) — diferença XOR `0x00001006` acompanha a topologia de cores.
  - `PROTCTRL`: `0x00000001` em repouso no K1 e no K3.

---

## 3. Registradores por LSID (`[P][O]`)

Cada contexto de sessão `lsid` (`0 <= lsid < NLSID`, ou seja, `0..3`) ocupa um bloco de `0x40` bytes a partir de:

```text
LSID_BASE(lsid) = 0xC050_0000 + 0x0200 + 0x40 * lsid
```

| LSID | Endereço Base |
|---:|---|
| 0 | `0xC050_0200` |
| 1 | `0xC050_0240` |
| 2 | `0xC050_0280` |
| 3 | `0xC050_02C0` |

### 3.1 Mapa de Offsets por LSID

| Offset | Nome | Acesso | Função e Campos |
|---:|---|---|---|
| `+0x00` | `CTRL` | R/W | `[7:0]` `DISALLOW` (bitmask de cores proibidos: bit `c = 1` proíbe core `c`); `[11:8]` `MAXCORES` (máximo de cores permitidos para o LSID) |
| `+0x04` | `MMU_CTRL` | R/W | PTE raiz (L0/L1) da tabela de páginas da sessão (mesmo formato de 32 bits dos PTEs da MMU, apontando para a página física L1 via `phys_cpu2vpu`) |
| `+0x08` | `NPROT` | R/W | Configuração de proteção/regiões protegidas da sessão (salvo/restaurado no PCB; valor em repouso observado = `1`) |
| `+0x0C` | `ALLOC` | R/W | Estado de alocação do LSID: `0` = `MVE_ALLOC_FREE`, `1` = `MVE_ALLOC_NON_PROTECTED`, `2` = `MVE_ALLOC_PROTECTED` |
| `+0x10` | `FLUSH_ALL` | W | Escrever `0` invalida todas as entradas da TLB/MMU deste LSID |
| `+0x14` | `SCHED` | R/W | `1` habilita o escalonamento deste LSID; `0` desabilita |
| `+0x18` | `TERMINATE` | R/W | Escrever `1` solicita término imediato do LSID; o hardware zera o registrador (`0`) quando o término conclui (poll obrigatório até `0`) |
| `+0x1C` | `LIRQVE` | R/W | Status local de IRQ do VPU → Host; escrever `0` (+ barreira `wmb()`) limpa a interrupção deste LSID |
| `+0x20` | `IRQHOST` | R/W | Escrever `1` sinaliza uma interrupção Host → Firmware para este LSID (também salvo/restaurado no PCB durante unmap/remap) |
| `+0x24` | `INTSIG` | R/W | Registrador de sinalização interna (sem documentação pública de bits; lê `0` em repouso) |
| `+0x2C` | `STREAMID` | R/W | Identificador de stream para SMMU/barramento (lê `0` em repouso) |
| `+0x30` | `BUSATTR_0` | R/W | Atributos AXI para páginas com `MVX_ATTR_PRIVATE` (`AT = 0`); default driver: `0x00` (DT) ou `0x33` observado |
| `+0x34` | `BUSATTR_1` | R/W | Atributos AXI para páginas com `MVX_ATTR_REFFRAME` (`AT = 1`); default driver: `0x00` (DT) ou `0x33` observado |
| `+0x38` | `BUSATTR_2` | R/W | Atributos AXI para páginas com `MVX_ATTR_SHARED_RO` (`AT = 2`); default driver: `0x33` |
| `+0x3C` | `BUSATTR_3` | R/W | Atributos AXI para páginas com `MVX_ATTR_SHARED_RW` (`AT = 3`); default driver: `0x33` |

### 3.2 Detalhe do Registrador `CTRL` (`+0x00`)

```text
 31             12 11       8 7              0
+-----------------+----------+----------------+
|    Reservado    | MAXCORES |    DISALLOW    |
+-----------------+----------+----------------+
```

- `MVE_CTRL_DISALLOW_SHIFT = 0`, `MVE_CTRL_DISALLOW_MASK = 0xFF`:
  - Para `ncores` físicos, os bits de cores inexistentes são marcados como proibidos: `disallow = (0xFFFFFFFF << ncores) & 0xFF`.
  - Exemplo K1 (`ncores = 2`): `disallow = 0xFC` (permite cores 0 e 1).
  - Exemplo K3 (`ncores = 4`): `disallow = 0xF0` (permite cores 0, 1, 2 e 3).
- `MVE_CTRL_MAXCORES_SHIFT = 8`, `MVE_CTRL_MAXCORES_MASK = 0xF`:
  - Configurado com o número máximo de cores que a sessão pode utilizar (`1..ncores`).

---

## 4. Snapshots Medidos em Hardware Real (Estado em Repouso) (`[O]`)

### 4.1 Registradores Globais

| Registrador | SpacemiT K1 (2 cores) | SpacemiT K3 (4 cores) |
|---|---|---|
| `HARDWARE_ID` | `0x56635002` | `0x56635002` |
| `ENABLE` | `0x00000000` | `0x00000000` |
| `NCORES` | `0x00000002` | `0x00000004` |
| `NLSID` | `0x00000004` | `0x00000004` |
| `CORELSID` | `0x000000FF` | `0x0000FFFF` |
| `JOBQUEUE` | `0x0F0F0F0F` | `0x0F0F0F0F` |
| `IRQVE` | `0x00000000` | `0x00000000` |
| `CLKFORCE` | `0x00000000` | `0x00000000` |
| `FUSE` | `0x00000000` | `0x00000000` |
| `CONFIG` | `0xFFFF6402` | `0xFFFF7404` |
| `PROTCTRL` | `0x00000001` | `0x00000001` |
| `RESET` | `0x00000000` | `0x00000000` |

### 4.2 Registradores por LSID (`lsid0..3` em repouso, K1 e K3)

```text
CTRL       = 0x00000000    SCHED      = 0x00000000    BUSATTR_0  = 0x00000033
MMU_CTRL   = 0x00000000    TERMINATE  = 0x00000000    BUSATTR_1  = 0x00000033
NPROT      = 0x00000001    LIRQVE     = 0x00000000    BUSATTR_2  = 0x00000033
ALLOC      = 0x00000000    IRQHOST    = 0x00000000    BUSATTR_3  = 0x00000033
FLUSH_ALL  = 0x00000000    INTSIG     = 0x00000000    STREAMID   = 0x00000000
```

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
