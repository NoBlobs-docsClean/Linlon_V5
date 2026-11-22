<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Toolchain Público RASC (`rasc-elf`) e Contrato ABI

Este documento explica como compilar um firmware *freestanding* em C e Assembly para as CPUs RASC do SpacemiT K1/K3 utilizando exclusivamente o toolchain público open-source da ARM, detalha a ABI gerada pelo compilador e documenta as limitações práticas conhecidas do toolchain.

---

## 1. Fonte Pública do Toolchain (`[P]`)

A ARM disponibiliza publicamente o toolchain completo para o alvo **`rasc-elf`**:

- **Repositório:** `https://github.com/ARM-software/rasc-devkit` (GPLv2)
- **Componentes:**
  - `binutils/` — GNU binutils **2.18** (`rasc-elf-as`, `rasc-elf-ld`, `rasc-elf-objdump`, `rasc-elf-objcopy`, `rasc-elf-ar`, `rasc-elf-ranlib`, `rasc-elf-nm`).
  - `gcc/` — GCC **4.2.3** com backend completo `gcc/config/rasc/` (`rasc.h`, `rasc.c`, `rasc.md`).
- **Licença:** GPLv2 (código gerado pela compilação do seu próprio código-fonte clean-room não herda licença do compilador; atenção apenas a não incluir runtime proprietário).

---

## 2. Compilação do Toolchain em Linux Moderno (x86-64) (`[P][O]`)

O `makefile` original do repositório pressupõe `gcc -m32`, `newlib` e `svn` (não presentes no repositório). O build manual direto em host 64 bits foi validado com sucesso (`[O]`):

### 2.1 Wrapper de Compatibilidade para GCC Hospedeiro Moderno

Compiladores modernos (GCC 14–16) tratam declarações implícitas e `-fno-common` como erro fatal no código de 2008. Crie um wrapper `cc-gnu89`:

```sh
#!/bin/sh
exec gcc -std=gnu89 -fcommon -Wno-error \
     -Wno-implicit-function-declaration \
     -Wno-builtin-declaration-mismatch "$@"
```

### 2.2 Build do `binutils` (`rasc-elf`)

```sh
mkdir -p build-binutils && cd build-binutils
CC=$PWD/../cc-gnu89 ../rasc-devkit/binutils/configure \
    --target=rasc-elf \
    --prefix=$PWD/../out \
    --program-prefix=rasc-elf- \
    --disable-nls --disable-werror \
    --disable-gdb --disable-sim --disable-gprof
make -j$(nproc)
make install
```

### 2.3 Build do `gcc` (`rasc-elf-gcc`, C freestanding)

Adicione `out/bin` ao `PATH` antes de compilar o GCC (necessário para que `libgcc.a` encontre `rasc-elf-ar` e `rasc-elf-ranlib`):

```sh
export PATH=$PWD/out/bin:$PATH
mkdir -p build-gcc && cd build-gcc
CC=$PWD/../cc-gnu89 ../rasc-devkit/gcc/configure \
    --target=rasc-elf \
    --prefix=$PWD/../out \
    --program-prefix=rasc-elf- \
    --enable-languages=c \
    --disable-nls --disable-bootstrap \
    --without-headers --with-newlib \
    --disable-libssp --disable-libgomp --disable-libmudflap \
    --disable-threads --disable-shared --disable-multilib \
    --with-as=$PWD/../out/bin/rasc-elf-as \
    --with-ld=$PWD/../out/bin/rasc-elf-ld
make -j$(nproc) all-gcc
make install-gcc
```

> **Dica de ambiente:** Se `/tmp` tiver espaço limitado, defina `TMPDIR` para um diretório com espaço suficiente antes de rodar `make` e `rasc-elf-gcc`.

---

## 3. Convenção de Chamada (ABI `rasc-elf-gcc`) (`[P][O]`)

Extraída de `gcc/config/rasc/rasc.h`/`rasc.c` e confirmada empiricamente em hardware real no K1:

| Elemento | Convenção |
|---|---|
| **Argumentos de função** | Primeiras **5 palavras (32 bits)** em **`r0` – `r4`**; argumentos adicionais (6º em diante) passados na **pilha** (`[sp, offset]`) |
| **Retorno (32 bits)** | **`r0`** |
| **Retorno (64 bits ou struct de 2 words)** | **`r0:r1`** (`r0` = palavra baixa, `r1` = palavra alta). *Nota medida:* retornar um `struct` de duas `u32` usa `r0:r1` sem instruções extras de empacotamento |
| **Caller-saved (voláteis)** | `r0` – `r4` |
| **Callee-saved (preservados)** | **`r5` – `r13`**, além de `r14` (`lr`) e `r15` (`sp`) |
| **Frame pointer (quando usado)** | `r13` |
| **Link register** | `r14` (`lr`) |
| **Stack pointer** | `r15` (`sp`), cresce para endereços decrescentes |
| **Prólogo típico** | `push rN-r14` (salva menor callee-saved usado até `lr`), `sub sp, #locals` |
| **Epílogo típico** | `add sp, #locals`, `popr rN-r14` (restaura `rN..r14` e retorna); funções folha retornam com `jmp lr` (pseudo-mnemônico `rts`) |
| **Chamada / Tail call** | `bsr simbolo` / `jsr rN`; tail calls usam `br simbolo` / `jmp rN` |
| **Ordem na pilha (`push rN-r14`)** | `rN` no **menor endereço**, `r14` (`lr`) no **maior endereço** |
| **Acesso a globais / strings** | Emitido automaticamente via `lpc rX, simbolo` (PC-relativo) |

---

## 4. Pipeline de Build de um Firmware Clean-Room `.fwb` (`[P][O]`)

Como o `rasc-devkit` não inclui `newlib` nem `crt0`, o firmware clean-room deve ser 100% **freestanding** (`-ffreestanding -nostdlib`):

```sh
# 1. Compilar startup em assembly e fontes C (C89 / GNU89)
rasc-elf-gcc -std=gnu89 -Os -ffreestanding -Wall -Wextra -c start.S -o start.o
rasc-elf-gcc -std=gnu89 -Os -ffreestanding -Wall -Wextra -c main.c -o main.o

# 2. Linkar com linker script que posiciona _start após o header de 180 bytes (ex.: 0xB4 ou 0xC0)
rasc-elf-ld -T firmware.ld -o firmware.elf start.o main.o

# 3. Extrair imagem flat (código + rodata)
rasc-elf-objcopy -O binary firmware.elf payload.bin

# 4. Empacotar no contêiner .fwb:
#    - 4 bytes de trampolim `br _start` em offset 0x00 (rasc_jmp)
#    - 176 bytes restantes de struct mvx_fw_header (protocol_major=3, protocol_minor=5,
#      text_length = offset_do_payload + len(payload.bin), bss_start_address, bss_bitmap, master_rw)
#    - padding até offset_do_payload + payload.bin
```

### Requisitos do Linker Script (`firmware.ld`)

1. Posicionar `.text.start` exatamente no offset de arquivo para o qual o trampolim `br` em `0x00` salta (ex.: `0xB4` ou `0xC0`, correspondendo a `VA 0x10B4` ou `0x10C0` na instância 0).
2. Alinhar `.bss` ao endereço `bss_start_address` declarado no header `.fwb` (múltiplo de `0x1000`, acima do fim de `.text`/`.rodata`).
3. Alinhar a seção compartilhada `.master_rw` ao endereço `master_rw_start_address` (múltiplo de `0x1000`), reservando páginas no `bss_bitmap_size`.
4. Definir o símbolo `stack_top` no topo da área de pilha dentro de `.bss` para carregamento via `lpc sp, stack_top` em `start.S`.
5. Se houver dados inicializados não-zero (`.data`), usar `AT(...)` em `.rodata` (LMA) com cópia no `start.S` para VMA em `.bss` usando endereçamento PC-relativo (`lpc`), pois o arquivo `.fwb` carrega apenas o segmento `text` (somente leitura/executável).

---

## 5. Armadilhas e Limitações Conhecidas do Toolchain (`[O]`)

1. **Segfault do assembler (`rasc-elf-as` 2.18) com endereços numéricos crus:**
   - Escrever `bsr 0x1234` ou `lpc r2, 0x44000` com número literal **causa segfault** no `as` 2.18.
   - **Solução:** Use **sempre rótulos/símbolos** (`bsr minha_funcao`, `lpc r2, meu_simbolo`). O compilador GCC sempre emite símbolos, logo apenas código escrito à mão em `.S` precisa observar essa regra.
2. **Dialeto C (GCC 4.2.3, 2008):**
   - Use `-std=gnu89` ou `-std=gnu99`. Recursos C11 (`_Static_assert`, `_Atomic`, `_Generic`) não existem.
   - Use `volatile` e barreiras de compilador (`__asm__ volatile ("" ::: "memory")`) em todo acesso MMIO (`0xFF00_0000+`) e filas compartilhadas (`0x1007_9000+`).
3. **Acesso a `CR`s em inline assembly:**
   - O índice do registrador de controle em `mtcr`/`mfcr` é um **imediato de instrução**, não um registrador. Portanto, funções helper C devem usar macros ou `switch` de constantes em tempo de compilação:
     ```c
     #define RASC_MTCR(cr_num, val) \
         __asm__ volatile ("mtcr %0, %1" :: "r"((unsigned int)(val)), "i"(cr_num) : "memory")
     #define RASC_MFCR(cr_num, out) \
         __asm__ volatile ("mfcr %0, %1" : "=r"(out) : "i"(cr_num))
     ```
4. **Pseudo-mnemônico `rts`:**
   - O assembler aceita `rts` como sinônimo de `jmp lr`; o `objdump` imprime `jmp lr`.
5. **Compatibilidade entre K1 e K3:**
   - O mesmo `rasc-elf-gcc` 4.2.3 gera código válido para K1 (2 cores) e K3 (4 cores), pois ambos compartilham a mesma ISA/ABI escalar RASC.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
