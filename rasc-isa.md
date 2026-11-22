<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# RASC ISA Reference (32-bit RISC)

This document specifies the Instruction Set Architecture (**ISA**) of the embedded **RASC** CPUs present in each core of the Arm China Linlon V5 VPU (`HARDWARE_ID = 0x56635002`). The encoding comes from ARM's public toolchain (`ARM-software/rasc-devkit`, GPLv2: `binutils/opcodes/rasc-opc.c` and `rasc-dis.c`), complemented by **empirical validation on real silicon** on SpacemiT K1.

---

## 1. Architecture Overview (`[P][O]`)

- **Word width:** 32-bit, **little-endian**.
- **Pipeline:** 5-stage RISC (optimized for high code density).
- **Variable instruction length:** Base word of **1 to 6 bytes** (scalar = 1 or 2 bytes; vector = 3 to 6 bytes), optionally followed by **1 to 4 byte suffixes** for large immediates and branch offsets. There is no 4-byte alignment requirement for instructions.
- **Cross-platform compatibility (`[O]`):** The same RASC ISA/ABI is used on SpacemiT K1 (Linlon V5, 2 cores) and SpacemiT K3 (Linlon V5, 4 cores).

### 1.1 Register File (`[P]`)

| Registers | Count | Role |
|---|---:|---|
| `r0` – `r13` | 14 | General-purpose scalar registers (32-bit) |
| `r14` (`lr`) | 1 | *Link Register* — receives the return address on `bsr`/`jsr` |
| `r15` (`sp`) | 1 | *Stack Pointer* — grows toward lower addresses |
| `v0` – `v15` | 16 | Vector/SIMD register file (`v*` / `vf*` extension) |
| `SR` | 1 | *Status Register* (accessed via `mfsr`/`mtsr`; IRQ control via `clri`/`seti`) |
| `CR0` – `CR255` | up to 256 | Control registers with an **8-bit** index (accessed via `mtcr`/`mfcr`) |

---

## 2. Bitwise Encoding Model (`[P]`)

Each descriptor in the public ISA table (`rasc-opc.c`, 123 descriptors) defines a bitmask written from **bit 0 (LSB)** to **MSB**, in multiples of 8 bits (1, 2, 3, 4, or 6 bytes):

| Mask Character | Meaning |
|---|---|
| `0` / `1` | Mandatory fixed bit |
| `p` | Sub-opcode field (1 to 3 bits) |
| `a` / `A` | Destination register `Rd` (lowercase: forbids `r15`; uppercase: allows `r0`–`r15`) |
| `b` / `B` | Source register `Rn` |
| `c` / `C` | Source register `Rm` |
| `d` / `D` | Source register `Ro` |
| `e` / `E` | `Rd` that encodes 1 extra opcode bit via comparison `(Rd > Rn)` → `op = 2*op + (Rd > Rn)` |
| `q` / `r` / `s` / `t` | Like `a`/`b`/`c`/`d`, forbidding `r14` and `r15` |
| `U` / `V` / `W` / `X` | Vector registers `v0`–`v15` |
| `i` | Fixed immediate (non-PC-relative, no suffix) |
| `m` | **Suffixable** immediate (high field values indicate appended bytes) |
| `j` | Second immediate (used in vector instructions) |
| `R` | Branch displacement / PC-relative address (suffixable) |
| `x` | *Don't care* |

### 2.1 Immediate Suffix Table (`m`) (`[P]`)

When the *n*-bit `m` field contains one of the escape values below, the actual value is read in **little-endian** from the bytes immediately following the base word (**without** applying factor or bias):

| `m` Field Width | No suffix (`value = field * factor + bias`) | 1 B Suffix | 2 B Suffix | 3 B Suffix | 4 B Suffix |
|---|---|---:|---:|---:|---:|
| 3 bits | `0..3` | `4` | `5` | `6` | `7` |
| 4 bits | `0..11` | `12` | `13` | `14` | `15` |
| 6 bits | `0..59` | `60` | `61` | `62` | `63` |

### 2.2 Branch / PC-Relative Suffix Table (`R`) (`[P]`)

| `R` Field Width | 1 B Suffix | 2 B Suffix | 3 B Suffix | 4 B Suffix |
|---|---:|---:|---:|---:|
| 2 bits (`br`, `bsr`, `lpc`) | `0` | `1` | `2` | `3` |
| 3 bits (BCC3) | `6` | `7` | — | — |
| 5 bits (`dbne`, `dbpl`) | `30` | `31` | — | — |
| 7 bits (`beq`, `bne` against 0) | `126` | `127` | — | — |

- **With suffix:** The branch suffix is **sign-extended** (two's complement) and added to the address immediately following the base instruction + immediate suffix (if any):
  ```text
  target = pc + base_size + imm_suffix_size + sign_extend(branch_suffix)
  ```
- **Without suffix (except BCC3):**
  ```text
  target = pc + R_field * factor + bias
  ```

### 2.3 Special BCC3 Semantics (Instruction Count) (`[P][O]`)

In conditional branches with a **3-bit** `R` field (families `00pppRRRmmmmaaaa`, `010ppRRRBBBBaaaa`, and `01110RRRbbbbeeee`), when the `R` field is in the range `0..5` (**no suffix**):

> **Normative rule (`[P][O]`):** The value `N = R_field` (or the count encoded by the bias) counts the number of **complete instructions to skip** after the branch instruction — **not** `N * 2` bytes!

Because RASC instructions have variable lengths (2 to 8 bytes), the hardware advances PC by skipping `N` decoded instructions according to the actual length of each skipped instruction. These semantics were confirmed both in the public `rasc-opc.c` source (`isa_bcc3_icnt_semantics() == 1`) and on real K1 silicon (skipping two 3-byte instructions in a single `bteq ... <skip 2>`).

---

## 3. Scalar Instruction Set (8- and 16-bit Base) (`[P]`)

| Mask (bit 0 → MSB) | Factor / Bias | Mnemonics | Operation |
|---|---:|---|---|
| `011110RR` | `1 / suffix` | `br` | Unconditional PC-relative branch (1 B base + 1–4 B suffix) |
| `011111RR` | `1 / suffix` | `bsr` | PC-relative subroutine call (`lr = pc_next`; branch) |
| `00pppRRRmmmmaaaa` | BCC3 | `beq, bne, blt, bge, blo, bhs, bteq, btne` | Compare `Rd` with `imm4` (signed: `lt/ge`; unsigned: `lo/hs`; bit test `(Rd & imm)==0/!=0`: `bteq/btne`) |
| `00pRRRRRaaaa1111` | `1 / -29` | `dbne, dbpl` | Decrement `Rd` (`Rd -= 1`) and branch if `!= 0` (`dbne`) or `>= 0` (`dbpl`) |
| `010ppRRRBBBBaaaa` | BCC3 | `bls, blo, ble, blt` | Compare `Rd` with `Rn` (unsigned `<=`, `<`; signed `<=`, `<`) |
| `0110pRRRRRRRAAAA` | `1 / -32` | `beq, bne` | Branch if `Rd == 0` or `Rd != 0` |
| `01110RRRbbbbeeee` | BCC3 | `beq, bne` | Compare `Rd` with `Rn` (`op` comes from `Rd > Rn`) |
| `0111000paaaa1111` | `1 / 0` | `pop, popr` | Restore `rN..r14` from stack; `popr` also executes return (`jmp lr`) |
| `01110010aaaa1111` | `1 / 0` | `push` | Save `rN..r14` on stack (`rN` at lowest address, `r14` at highest) |
| `10p0mmmmaaaaBBBB` | `4 / 0` | `ld, st` | 32-bit load/store at `[Rn + imm*4]` (or `[Rn + suffix]`) |
| `10p10mmmaaaaBBBB` | `2 / 0` | `ldh, sth` | 16-bit load/store (zero-extended) at `[Rn + imm*2]` |
| `10p11mmmaaaaBBBB` | `1 / 0` | `ldb, stb` | 8-bit load/store (zero-extended) at `[Rn + imm]` |
| `11000pppmmmmaaaa` | `1 / 0` | `add, sub, adc, sbc, and, or, xor, andn` | `Rd = Rd <op> imm4` (`andn` = `Rd & ~imm`) |
| `11000pmmmmmm1111` | `4 / 0` | `add, sub` | `sp = sp +/- imm6*4` (stack frame adjustment) |
| `11001pppmmmmaaaa` | `1 / 0` | `rsb, lsl, lsr, asr` | `rsb`: `Rd = imm - Rd`; logical/arithmetic shifts by `imm4` |
| `11001xRRAAAA1111` | `1 / suffix` | `lpc` | Load PC-relative address into `Rd` (essential for multi-instance PIC) |
| `11010pppbbbbaaaa` | `1 / 0` | `add, sub, adc, sbc, and, or, xor, andn` | `Rd = Rd <op> Rn` |
| `1101000paaaa1111` | `1 / 0` | `jmp, jsr` | Indirect branch or call via register (`jmp lr` = return `rts`) |
| `1101001pmmmm1111` | `1 / 0` | `jmp, jsr` | Branch or call to immediate address |
| `1101000p11111111` | `1 / 0` | `rte, wait` | `rte`: return from exception/interrupt; `wait`: halt until interrupt |
| `110101ppaaaa1111` | `1 / 0` | `mfsr, mtsr, clri, seti` | Read/write `SR`; disable/enable interrupts |
| `11010ppp1111aaaa` | `1 / 0` | `not, rrx, sextb, zextb, sexth, zexth` | Unary operations (`not` = `~Rd`; 8- and 16-bit sign/zero extensions) |
| `11011pppbbbbaaaa` | `1 / 0` | `rsb, lsl, lsr, asr, mul` | `Rd = Rn - Rd`, shifts by `Rn`, 32×32→32 multiply (`mul`) |
| `11011mmmaaaa1111` | `1 / 0` | `mtcr` | Write `Rd` to control register `CR[imm]` (8-bit `imm` via suffix) |
| `11011mmm1111aaaa` | `1 / 0` | `mfcr` | Read control register `CR[imm]` into `Rd` (8-bit `imm` via suffix) |
| `111000ppBBBBAAAA` | `1 / 0` | `mov, movh, movw` | `mov`: `Rd = Rn`; `movh`: `Rd = Rn << 1`; `movw`: `Rd = Rn << 2` |
| `1110010pmmmmAAAA` | `1 / 0` | `mov, movn` | `mov`: `Rd = imm`; `movn`: `Rd = ~imm` (bitwise NOT) |

---

## 4. Critical Semantics Validated on K1 Silicon (`[O]`)

Tests executed directly on the SpacemiT K1 RASC CPUs confirmed exact semantics that often mislead readers who judge superficially from the mnemonics:

| Instruction | Hardware-Proven Semantics | Common Pitfall to Avoid |
|---|---|---|
| `movh rd, rn` | **`rd = rn << 1`** (*halfword* index scaling, 2 bytes) | It is **not** `rn << 16`! (Tested: `0x1234 → 0x2468`) |
| `movw rd, rn` | **`rd = rn << 2`** (*word* index scaling, 4 bytes) | It is **not** 16-bit zero-extend! (Tested: `0xABCD → 0x2AF34`) |
| `movn rd, imm` | **`rd = ~imm`** (bitwise complement / NOT) | It is **not** arithmetic negation `0 - imm`! (e.g., `movn r, 0 = 0xFFFFFFFF`; `movn r, 0xFD4FFF = 0xFF02B000`) |
| `add` / `adc` | `add` sets the *carry* flag; `adc` adds the *carry* | Confirmed: `0xFFFFFFFF + 1 = 0` with `carry = 1`; subsequent `adc 0` produces `1` |
| `push rN-r14` | Decrements `sp` by `4 * (15 - N)` and stores `rN` at the **lowest** address through `r14` (`lr`) at the **highest** address | Order matches the exception frame convention; `pop`/`popr` reverses the operation |
| `mtcr` / `mfcr` | The CR index is an **8-bit integer (`0x00..0xFF`)**: values `0..3` fit in the 3-bit field and `4..255` use a 1-byte suffix (`m = 4`) | **Do not** mask the index to 4 bits (`& 0xF`): on silicon, `CR 0x94` and `CR 4` have distinct effects! |
| `rrx rd` | **Open / anomalous (`[O]`):** With confirmed `carry = 1`, `rrx` on `0x80000001` left the register unchanged on hardware | Avoid relying on `rrx` until further characterization; not emitted by GCC in standard C code |

---

## 5. Vector / SIMD Extension (`v0`–`v15`) (`[P]`)

The public ISA table (`rasc-opc.c`) also defines 24- to 48-bit vector instructions (prefix `1111` in byte 0), including:
- **Vector load/store:** `vfld`, `vfst`, `vfldb`, `vfstb` (with and without post-increment `[++]`).
- **Transfer and rearrangement:** `vmov`, `vmovb`, `vrep`, `vint`, `vdint`, `vsel0..3`, `vlrot`, `vrot`, `vmir`, `vtrans`.
- **Arithmetic and saturation:** `vadd`, `vsub`, `vadd1`, `vadc`, `vsbc`, `vmax`, `vmin`, `vsat`, `vclip`, `vneg`, `vnot`, `vsext`, `vzext` (signed/unsigned saturating `ss`/`us` and byte `b` variants).
- **Multiply/MAC:** `vmul`, `vmac`, `vmull`, `vmacl` and saturating/rounding variants.
- **Logic and shifts:** `vand`, `vor`, `vxor`, `vandn`, `vlsl`, `vlsr`, `vasr`.
- **Compare/skip and synchronization:** `vsbm`, `vseq`, `vsne`, `vsteq`, `vstne`, `vslt`, `vsgt`, `vslo`, `vshi`, `vsle`, `vsge`, `vsls`, `vshs`, `vsync`, `vend`.

> **Note for the clean-room team (`[O]`):** The `rasc-elf-gcc` C compiler does **not** generate `v*` vector instructions, and the firmware control plane requires no vector instructions. Detailed per-lane semantics of `v*` instructions have not yet been verified on silicon and remain outside the minimum set required for M1–M3.

---

*License: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
