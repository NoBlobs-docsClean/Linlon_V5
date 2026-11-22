<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Referência da ISA RASC (32-bit RISC)

Este documento especifica a arquitetura de conjunto de instruções (**ISA**) das CPUs embarcadas **RASC** presentes em cada core da VPU Arm China Linlon V5 (`HARDWARE_ID = 0x56635002`). A codificação provém do toolchain público da ARM (`ARM-software/rasc-devkit`, GPLv2: `binutils/opcodes/rasc-opc.c` e `rasc-dis.c`), complementada por **validação empírica em silício real** no SpacemiT K1.

---

## 1. Visão Geral da Arquitetura (`[P][O]`)

- **Largura de palavra:** 32 bits, **little-endian**.
- **Pipeline:** RISC de 5 estágios (otimizado para alta densidade de código).
- **Comprimento de instrução variável:** Palavra base de **1 a 6 bytes** (escalares = 1 ou 2 bytes; vetoriais = 3 a 6 bytes), opcionalmente seguida de **sufixos de 1 a 4 bytes** para imediatos grandes e deslocamentos de branch. Não há requisito de alinhamento de 4 bytes para instruções.
- **Compatibilidade entre plataformas (`[O]`):** A mesma ISA/ABI RASC é utilizada no SpacemiT K1 (Linlon V5, 2 cores) e no SpacemiT K3 (Linlon V5, 4 cores).

### 1.1 Banco de Registradores (`[P]`)

| Registradores | Quantidade | Papel |
|---|---:|---|
| `r0` – `r13` | 14 | Registradores escalares de uso geral (32 bits) |
| `r14` (`lr`) | 1 | *Link Register* — recebe o endereço de retorno em `bsr`/`jsr` |
| `r15` (`sp`) | 1 | *Stack Pointer* — ponteiro de pilha (cresce para endereços menores) |
| `v0` – `v15` | 16 | Banco de registradores vetoriais/SIMD (extensão `v*` / `vf*`) |
| `SR` | 1 | *Status Register* (acessado por `mfsr`/`mtsr`; controle de IRQ por `clri`/`seti`) |
| `CR0` – `CR255` | até 256 | Registradores de controle com índice de **8 bits** (acessados por `mtcr`/`mfcr`) |

---

## 2. Modelo de Codificação Bit a Bit (`[P]`)

Cada descritor na tabela pública da ISA (`rasc-opc.c`, 123 descritores) define uma máscara de bits escrita do **bit 0 (LSB)** ao **MSB**, em múltiplos de 8 bits (1, 2, 3, 4 ou 6 bytes):

| Caractere na Máscara | Significado |
|---|---|
| `0` / `1` | Bit fixo obrigatório |
| `p` | Campo de sub-opcode (1 a 3 bits) |
| `a` / `A` | Registrador destino `Rd` (minúscula: proíbe `r15`; maiúscula: permite `r0`–`r15`) |
| `b` / `B` | Registrador fonte `Rn` |
| `c` / `C` | Registrador fonte `Rm` |
| `d` / `D` | Registrador fonte `Ro` |
| `e` / `E` | `Rd` que codifica 1 bit extra de opcode via comparação `(Rd > Rn)` → `op = 2*op + (Rd > Rn)` |
| `q` / `r` / `s` / `t` | Como `a`/`b`/`c`/`d`, proibindo `r14` e `r15` |
| `U` / `V` / `W` / `X` | Registradores vetoriais `v0`–`v15` |
| `i` | Imediato fixo (não PC-relativo, sem sufixo) |
| `m` | Imediato **sufixável** (valores altos do campo indicam bytes anexados) |
| `j` | Segundo imediato (usado em instruções vetoriais) |
| `R` | Deslocamento de branch / endereço PC-relativo (sufixável) |
| `x` | *Don't care* |

### 2.1 Tabela de Sufixos de Imediato (`m`) (`[P]`)

Quando o campo `m` de largura *n* bits contém um dos valores de escape abaixo, o valor real é lido em **little-endian** dos bytes imediatamente seguintes à palavra base (**sem** aplicar fator ou bias):

| Largura do campo `m` | Sem sufixo (`valor = campo * fator + bias`) | Sufixo 1 B | Sufixo 2 B | Sufixo 3 B | Sufixo 4 B |
|---|---|---:|---:|---:|---:|
| 3 bits | `0..3` | `4` | `5` | `6` | `7` |
| 4 bits | `0..11` | `12` | `13` | `14` | `15` |
| 6 bits | `0..59` | `60` | `61` | `62` | `63` |

### 2.2 Tabela de Sufixos de Branch / PC-Relativo (`R`) (`[P]`)

| Largura do campo `R` | Sufixo 1 B | Sufixo 2 B | Sufixo 3 B | Sufixo 4 B |
|---|---:|---:|---:|---:|
| 2 bits (`br`, `bsr`, `lpc`) | `0` | `1` | `2` | `3` |
| 3 bits (BCC3) | `6` | `7` | — | — |
| 5 bits (`dbne`, `dbpl`) | `30` | `31` | — | — |
| 7 bits (`beq`, `bne` contra 0) | `126` | `127` | — | — |

- **Com sufixo:** O sufixo de branch é **sign-extended** (complemento de dois) e somado ao endereço imediatamente após a instrução base + sufixo de imediato (se houver):
  ```text
  alvo = pc + tamanho_base + tamanho_sufixo_imm + sign_extend(sufixo_branch)
  ```
- **Sem sufixo (exceto BCC3):**
  ```text
  alvo = pc + campo_R * fator + bias
  ```

### 2.3 Semântica Especial BCC3 (Contagem de Instruções) (`[P][O]`)

Nos branches condicionais de campo `R` de **3 bits** (famílias `00pppRRRmmmmaaaa`, `010ppRRRBBBBaaaa` e `01110RRRbbbbeeee`), quando o campo `R` está no intervalo `0..5` (**sem sufixo**):

> **Regra normativa (`[P][O]`):** O valor `N = campo_R` (ou a contagem codificada pelo bias) conta o número de **instruções completas a pular** após a instrução de branch — e **não** `N * 2` bytes!

Como instruções RASC têm comprimentos variáveis (2 a 8 bytes), o hardware avança o PC saltando `N` instruções decodificadas de acordo com o tamanho real de cada instrução pulada. Essa semântica foi confirmada tanto no código público `rasc-opc.c` (`isa_bcc3_icnt_semantics() == 1`) quanto em silício real no K1 (pulando duas instruções de 3 bytes em um único `bteq ... <skip 2>`).

---

## 3. Conjunto de Instruções Escalares (8 e 16 bits Base) (`[P]`)

| Máscara (bit 0 → MSB) | Fator / Bias | Mnemônicos | Operação |
|---|---:|---|---|
| `011110RR` | `1 / sufixo` | `br` | Salto incondicional PC-relativo (base 1 B + sufixo 1–4 B) |
| `011111RR` | `1 / sufixo` | `bsr` | Chamada de sub-rotina PC-relativa (`lr = pc_next`; salto) |
| `00pppRRRmmmmaaaa` | BCC3 | `beq, bne, blt, bge, blo, bhs, bteq, btne` | Compara `Rd` com `imm4` (signed: `lt/ge`; unsigned: `lo/hs`; teste de bits `(Rd & imm)==0/!=0`: `bteq/btne`) |
| `00pRRRRRaaaa1111` | `1 / -29` | `dbne, dbpl` | Decrementa `Rd` (`Rd -= 1`) e salta se `!= 0` (`dbne`) ou `>= 0` (`dbpl`) |
| `010ppRRRBBBBaaaa` | BCC3 | `bls, blo, ble, blt` | Compara `Rd` com `Rn` (unsigned `<=`, `<`; signed `<=`, `<`) |
| `0110pRRRRRRRAAAA` | `1 / -32` | `beq, bne` | Salta se `Rd == 0` ou `Rd != 0` |
| `01110RRRbbbbeeee` | BCC3 | `beq, bne` | Compara `Rd` com `Rn` (`op` vem de `Rd > Rn`) |
| `0111000paaaa1111` | `1 / 0` | `pop, popr` | Restaura `rN..r14` da pilha; `popr` também executa retorno (`jmp lr`) |
| `01110010aaaa1111` | `1 / 0` | `push` | Salva `rN..r14` na pilha (`rN` no menor endereço, `r14` no maior) |
| `10p0mmmmaaaaBBBB` | `4 / 0` | `ld, st` | Load/store 32 bits em `[Rn + imm*4]` (ou `[Rn + sufixo]`) |
| `10p10mmmaaaaBBBB` | `2 / 0` | `ldh, sth` | Load/store 16 bits (zero-extended) em `[Rn + imm*2]` |
| `10p11mmmaaaaBBBB` | `1 / 0` | `ldb, stb` | Load/store 8 bits (zero-extended) em `[Rn + imm]` |
| `11000pppmmmmaaaa` | `1 / 0` | `add, sub, adc, sbc, and, or, xor, andn` | `Rd = Rd <op> imm4` (`andn` = `Rd & ~imm`) |
| `11000pmmmmmm1111` | `4 / 0` | `add, sub` | `sp = sp +/- imm6*4` (ajuste de frame de pilha) |
| `11001pppmmmmaaaa` | `1 / 0` | `rsb, lsl, lsr, asr` | `rsb`: `Rd = imm - Rd`; shifts lógico/aritmético por `imm4` |
| `11001xRRAAAA1111` | `1 / sufixo` | `lpc` | Carrega endereço PC-relativo em `Rd` (essencial para PIC multi-instância) |
| `11010pppbbbbaaaa` | `1 / 0` | `add, sub, adc, sbc, and, or, xor, andn` | `Rd = Rd <op> Rn` |
| `1101000paaaa1111` | `1 / 0` | `jmp, jsr` | Salto ou chamada indireta por registrador (`jmp lr` = retorno `rts`) |
| `1101001pmmmm1111` | `1 / 0` | `jmp, jsr` | Salto ou chamada para endereço imediato |
| `1101000p11111111` | `1 / 0` | `rte, wait` | `rte`: retorno de exceção/interrupção; `wait`: halt até interrupção |
| `110101ppaaaa1111` | `1 / 0` | `mfsr, mtsr, clri, seti` | Lê/escreve `SR`; desabilita/habilita interrupções |
| `11010ppp1111aaaa` | `1 / 0` | `not, rrx, sextb, zextb, sexth, zexth` | Operações unárias (`not` = `~Rd`; extensões de sinal/zero 8 e 16 bits) |
| `11011pppbbbbaaaa` | `1 / 0` | `rsb, lsl, lsr, asr, mul` | `Rd = Rn - Rd`, shifts por `Rn`, multiplicação 32×32→32 (`mul`) |
| `11011mmmaaaa1111` | `1 / 0` | `mtcr` | Escreve `Rd` no registrador de controle `CR[imm]` (`imm` de 8 bits via sufixo) |
| `11011mmm1111aaaa` | `1 / 0` | `mfcr` | Lê registrador de controle `CR[imm]` para `Rd` (`imm` de 8 bits via sufixo) |
| `111000ppBBBBAAAA` | `1 / 0` | `mov, movh, movw` | `mov`: `Rd = Rn`; `movh`: `Rd = Rn << 1`; `movw`: `Rd = Rn << 2` |
| `1110010pmmmmAAAA` | `1 / 0` | `mov, movn` | `mov`: `Rd = imm`; `movn`: `Rd = ~imm` (NOT bit a bit) |

---

## 4. Semânticas Críticas Validadas no Silício do K1 (`[O]`)

Testes executados diretamente nas CPUs RASC do SpacemiT K1 confirmaram semânticas exatas que frequentemente induzem a erro por leitura superficial dos mnemônicos:

| Instrução | Semântica Comprovada em Hardware | Armadilha Comum a Evitar |
|---|---|---|
| `movh rd, rn` | **`rd = rn << 1`** (escala de índice de *halfword*, 2 bytes) | **Não** é `rn << 16`! (Testado: `0x1234 → 0x2468`) |
| `movw rd, rn` | **`rd = rn << 2`** (escala de índice de *word*, 4 bytes) | **Não** é zero-extend 16 bits! (Testado: `0xABCD → 0x2AF34`) |
| `movn rd, imm` | **`rd = ~imm`** (complemento bit a bit / NOT) | **Não** é negação aritmética `0 - imm`! (Ex.: `movn r, 0 = 0xFFFFFFFF`; `movn r, 0xFD4FFF = 0xFF02B000`) |
| `add` / `adc` | `add` gera flag de *carry*; `adc` soma o *carry* | Confirmado: `0xFFFFFFFF + 1 = 0` com `carry = 1`; `adc 0` subsequente produz `1` |
| `push rN-r14` | Decrementa `sp` em `4 * (15 - N)` e grava `rN` no **menor** endereço até `r14` (`lr`) no **maior** endereço | Ordem idêntica à convenção de frame de exceção; `pop`/`popr` inverte a operação |
| `mtcr` / `mfcr` | O índice do CR é um **inteiro de 8 bits (`0x00..0xFF`)**: valores `0..3` cabem no campo de 3 bits e `4..255` usam sufixo de 1 byte (`m = 4`) | **Não** mascarar o índice para 4 bits (`& 0xF`): em silício, `CR 0x94` e `CR 4` têm efeitos distintos! |
| `rrx rd` | **Em aberto / anômala (`[O]`):** Com `carry = 1` confirmado, `rrx` sobre `0x80000001` deixou o registrador inalterado em hardware | Evitar depender de `rrx` até caracterização adicional; não é emitida pelo GCC em código C padrão |

---

## 5. Extensão Vetorial / SIMD (`v0`–`v15`) (`[P]`)

A tabela pública da ISA (`rasc-opc.c`) define também instruções vetoriais de 24 a 48 bits (prefixo `1111` no byte 0), incluindo:
- **Carga/armazenamento vetorial:** `vfld`, `vfst`, `vfldb`, `vfstb` (com e sem pós-incremento `[++]`).
- **Transferência e rearranjo:** `vmov`, `vmovb`, `vrep`, `vint`, `vdint`, `vsel0..3`, `vlrot`, `vrot`, `vmir`, `vtrans`.
- **Aritmética e saturação:** `vadd`, `vsub`, `vadd1`, `vadc`, `vsbc`, `vmax`, `vmin`, `vsat`, `vclip`, `vneg`, `vnot`, `vsext`, `vzext` (variantes signed/unsigned saturating `ss`/`us` e byte `b`).
- **Multiplicação/MAC:** `vmul`, `vmac`, `vmull`, `vmacl` e variantes saturantes/arredondadas.
- **Lógica e shifts:** `vand`, `vor`, `vxor`, `vandn`, `vlsl`, `vlsr`, `vasr`.
- **Comparação/skip e sincronização:** `vsbm`, `vseq`, `vsne`, `vsteq`, `vstne`, `vslt`, `vsgt`, `vslo`, `vshi`, `vsle`, `vsge`, `vsls`, `vshs`, `vsync`, `vend`.

> **Nota para o time limpo (`[O]`):** O compilador C `rasc-elf-gcc` **não** gera instruções vetoriais `v*`, e o plano de controle do firmware não exige nenhuma instrução vetorial. A semântica detalhada por lane das instruções `v*` ainda não foi verificada em silício e permanece fora do conjunto mínimo necessário para M1–M3.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
