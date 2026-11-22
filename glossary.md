<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Glossário — VPU SpacemiT K1 / K3 (Arm China Linlon V5)

| Sigla / Termo | Definição |
|---|---|
| **AFBC** | *Arm Frame Buffer Compression* — formato de compressão de framebuffers sem perdas em superblocos (16×16 ou 32×8) utilizado pelo VDMA e pelo subsistema REF para reduzir largura de banda DDR. |
| **APB3** | *AMBA Advanced Peripheral Bus 3* — barramento de controle usado pela CPU RISC-V do SoC para acessar os registradores globais e por LSID em `0xC050_0000`. |
| **AXI** | *AMBA Advanced eXtensible Interface* — barramento de dados de 128 bits usado pela VPU para ler/escrever DRAM (bitstreams, framebuffers, `IntBuf` e tabelas da MMU). |
| **BCC3** | Família de instruções de desvio condicional RASC com campo de alcance de 3 bits cujo valor sem sufixo conta **instruções** (de comprimento variável) a pular, e não bytes. |
| **BITSTREAM** | Bloco acelerador interno de entropia/bitstream (CABAC, CAVLC, decodificação booleana VP8/VP9 e predição de MV). |
| **BSS (`bss_bitmap`)** | Segmento de memória RW zerado pelo driver na carga do `.fwb`; páginas privadas são alocadas por instância de core conforme o bitmap do header. |
| **Clean-Room** | Metodologia de sala limpa onde a equipe de implementação trabalha exclusivamente sobre especificações comportamentais e contratos públicos, sem contato com binários ou código proprietário. |
| **CR (`CR0`–`CR255`)** | *Control Registers* das CPUs RASC, endereçados por índice de 8 bits via instruções `mtcr`/`mfcr`. |
| **CTU** | *Coding Tree Unit* — unidade básica de codificação em HEVC (16×16, 32×32 ou 64×64). |
| **DBL** | *Deblocking Filter* — bloco interno de filtragem em laço (deblocking e SAO). |
| **DPB** | *Decoded Picture Buffer* — pool lógico de quadros decodificados mantidos para referência e reordenação de exibição. |
| **DSL** | *Downscale* — redimensionamento flexível na saída do decodificador (`DEC_DOWNSCALE`). |
| **DVFS** | *Dynamic Voltage and Frequency Scaling* — ajuste dinâmico da frequência de `vpu_clk` (307,2–819,2 MHz no K1; até 1 GHz no K3). |
| **FAR** | *Fault Address Register* — endereço virtual faltoso capturado em `CR2` durante uma exceção de memória / `MMU ABORT`. |
| **`.fwb`** | *Firmware Binary* — contêiner plano (header `mvx_fw_header` de 180 bytes + imagem de código/rodata) carregado pelo driver `mvx`. |
| **IntBuf** | *Intermediate Buffer* — buffers intermediários em DDR (alocados via RPC) que desacoplam o estágio de *parse* (`mbinfo`/residuais) do estágio de *pipe* (`MC`/`TRF`/`DBL`). |
| **JOBQUEUE** | Registrador global `0x14` com 4 slots de 8 bits `{ncores-1[7:4], lsid[3:0]}` usado pelo host para submeter sessões ao escalonador de hardware. |
| **Linlon V5** | Marca comercial da Arm China para a geração de VPU derivada da família Mali-V52/V76 (`HARDWARE_ID = 0x56635002`, `compatible = "arm china,linlon-v5"`). |
| **LSID** | *Local Session ID* — contexto de hardware independente de sessão (4 disponíveis: LSID 0–3), com MMU, IRQs e controle de cores próprios. |
| **LTR** | *Long Term Reference* — quadro de referência de longo prazo em codificação H.264/HEVC. |
| **`master_rw`** | Região de páginas RW declarada no header `.fwb` cujas páginas físicas são **compartilhadas** no mesmo offset virtual entre todos os cores de uma sessão multi-core. |
| **MC / ME** | *Motion Compensation* / *Motion Estimation* — blocos internos de compensação e busca de movimento. |
| **MMU** | *Memory Management Unit* — unidade de tradução de 2 níveis da VPU (VA 32 bits → PA 40 bits, páginas de 4 KiB, PTE de 32 bits). |
| **MVE 3.5** | *Mali/Linlon Video Engine Protocol v3.5* — protocolo de filas circulares, mensagens, buffers e RPC entre driver Linux e firmware. |
| **`mvx`** | Driver Linux GPL-2.0 V4L2 Mem2Mem da Arm China / SpacemiT para a família Linlon/Mali-V. |
| **PCB** | *Process Control Block* (`struct mvx_lsid_pcb`) — conjunto de registradores de LSID (`ctrl`, `mmu_ctrl`, `irqhost`, `nprot`) salvo pelo host ao desmapear uma sessão ociosa. |
| **`ram_print`** | Região de 512 KiB em `0x1010_0000` usada como log circular de texto do firmware para o host (magic `0x11223356`). |
| **RASC** | Microprocessador RISC de 32 bits (pipeline de 5 estágios, instruções de comprimento variável) projetado pela ARM para controle de VPUs Mali-V/Linlon; existem 5 por core. |
| **REF** | Subsistema interno de gerenciamento e cache de quadros de referência (tipicamente em AFBC). |
| **RPC** | *Remote Procedure Call* — canal síncrono em `0x1007_F000` usado pelo firmware para solicitar `DEBUG_PRINTF`, `MEM_ALLOC`, `MEM_RESIZE` e `MEM_FREE` ao host. |
| **SAO** | *Sample Adaptive Offset* — filtro em laço do padrão HEVC executado após o deblocking. |
| **SDMA / VDMA** | *System DMA* (metadados, `mbinfo`, linhas de contexto) e *Video DMA* (stripes de pixels, AFBC, rotação, downscale). |
| **SR** | *Status Register* da CPU RASC (lido/escrito por `mfsr`/`mtsr`). |
| **TRF** | *Transform* — bloco interno de transformada (DCT/DST/integer) e quantização/dequantização. |
| **V4L2 M2M** | *Video4Linux2 Memory-to-Memory* — interface Linux exposta pelo driver `mvx` (`/dev/video*`) para aplicações de vídeo. |

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
