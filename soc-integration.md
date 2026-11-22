<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Integração no SoC — SpacemiT K1 e SpacemiT K3

Este documento descreve como a VPU Linlon V5 se integra aos SoCs **SpacemiT K1** e **SpacemiT K3**: nós de Device Tree, recursos de plataforma, clocks, reset de hardware, gerenciamento de energia (PM/DVFS) e a **tradução obrigatória de endereços físicos DMA** do K1.

---

## 1. Recursos de Plataforma e Device Tree (`[P][O]`)

### 1.1 SpacemiT K1 (`k1-x.dtsi`)

```dts
linlon-v5@c0500000 {
    compatible = "arm china,linlon-v5";
    reg = <0x0 0xC0500000 0x0 0x10000>;     /* 64 KiB mapeados no DT (512 KiB no mapa do SoC) */
    interrupt-parent = <&intc>;
    interrupts = <74>;
    clocks = <&ccu CLK_VPU>;
    clock-names = "vpu_clk";
    power-domains = <&power K1X_PMU_VPU_PWR_DOMAIN>;
    resets = <&reset RESET_VPU>;
    interconnects = <&dram_range1>;
    interconnect-names = "dma-mem";
    clk,pm-runtime,no-sleep;
    status = "okay";
};
```

### 1.2 Tabela Comparativa de Recursos (K1 × K3)

| Recurso | SpacemiT K1 | SpacemiT K3 | Proveniência |
|---|---|---|---|
| `compatible` | `"arm china,linlon-v5"` | `"arm china,linlon-v5"` | `[P][O]` |
| Base MMIO (APB) | `0x0000_0000_C050_0000` | `0x0000_0000_C050_0000` | `[P][O]` |
| Janela reservada no mapa do SoC | `512 KiB` (`0x80000`) | `512 KiB` (`0x80000`) | `[P]` |
| Janela `reg` no DT | `64 KiB` (`0x10000`) | `8 KiB` (`0x2000`) | `[P][O]` |
| Interrupção (IRQ) | `74` (`0x4A`, `sys_int_ap[74]`) | `74` (`0x4A`, level-high) | `[P][O]` |
| Flag de registro IRQ | `IRQF_SHARED` | `IRQF_SHARED` | `[P]` |
| Clock | `CLK_VPU` (`vpu_clk`) | `vpu_clk` (pai `pll2_d3`) | `[P][O]` |
| Reset | `RESET_VPU` | Reset de domínio VPU | `[P]` |
| Domínio de energia | `K1X_PMU_VPU_PWR_DOMAIN` | Domínio PMU VPU | `[P]` |
| Máscara DMA do dispositivo | `DMA_BIT_MASK(34)` (34 bits) | ≥ 34 bits | `[P]` |
| Propriedade opcional `busattr` | Ausente no DT (usa default do driver) | `0x33` observado em `BUSATTR_0..3` | `[P][O]` |

> **Nota sobre compatíveis legados (`[P]`):** O driver `mvx` também declara compatíveis legados (`arm,mali-mve`, `arm,mali-v500`, `arm,mali-v550`, `arm,mali-v61`, `arm china,linlon-v7`), mas tanto K1 quanto K3 usam `"arm china,linlon-v5"`.

---

## 2. Sequência de Inicialização (Probe) e Reset do Hardware (`[P]`)

### 2.1 Sequência de Probe do Host

1. Mapear a região MMIO (`0xC050_0000`) e obter a IRQ `74`.
2. Configurar máscara DMA coerente e de streaming para **34 bits** (`DMA_BIT_MASK(34)`).
3. Habilitar `pm_runtime` e adquirir o clock `vpu_clk` e o controle de reset `RESET_VPU`.
4. Retirar o bloco de reset (`reset_control_deassert`).
5. Configurar a frequência nominal do clock:
   - **K1:** `819.200.000 Hz` (`819,2 MHz`).
   - **K3:** `1.000.000.000 Hz` (`1 GHz`).
6. Ler `HARDWARE_ID` (`0x00`), validar família (`0x5662`/`0x5663` → `MVE_v52_v76`), ler `FUSE` (`0x34`), `NCORES` (`0x08`) e `NLSID` (`0x0C`).
7. Inicializar estruturas de escalonador/LSID e registrar o tratador de interrupção (`IRQF_SHARED`) com workqueue para o *bottom half*.

### 2.2 Sequência de Reset Interno do Bloco (`reset_hw`)

Executada no religamento (*runtime resume* / *system resume*) antes de reativar o escalonador:

1. Ler `ncores = readl(BASE + 0x08)` (`NCORES`).
2. Montar a máscara esperada de cores livres (`corelsid_mask`): um nibble `0xF` para cada core ativo:
   - K1 (`ncores = 2`): `corelsid_mask = 0x0000_00FF`.
   - K3 (`ncores = 4`): `corelsid_mask = 0x0000_FFFF`.
3. Escrever `1` em `RESET` (`BASE + 0x50`).
4. Fazer poll em `CORELSID` (`BASE + 0x10`) até que `readl(CORELSID) == corelsid_mask` (garantindo que todos os cores saíram de execução e retornaram ao estado livre `0xF`).
5. Escrever `0` em `CLKFORCE` (`BASE + 0x24`) para liberar o *clock gating* automático do bloco.

> **Pré-condição crítica (`[P]`):** O reset só é considerado completo quando **todos** os nibbles de `CORELSID` correspondentes aos `NCORES` físicos leem `0xF`.

---

## 3. Clocks, Energia e DVFS (`[P][O]`)

### 3.1 Tabela de Frequências (OPPs)

| Nível (`vmin`) | Frequência (Hz) | Disponível no K1 | Disponível no K3 |
|---:|---:|:---:|:---:|
| 0 | `307.200.000` (307,2 MHz) | Sim | Sim |
| 1 | `409.600.000` (409,6 MHz) | Sim | Sim |
| 2 | `491.520.000` (491,52 MHz) | Sim | Sim |
| 3 | `600.000.000` (600,0 MHz) | Sim | Sim |
| 4 | `614.400.000` (614,4 MHz) | Sim | Sim |
| 5 | `750.000.000` (750,0 MHz) | Sim | Sim |
| 6 | **`819.200.000` (819,2 MHz)** | **Sim (máx. K1)** | Sim |
| 7 | **`1.000.000.000` (1,0 GHz)** | Não | **Sim (máx. K3)** |

### 3.2 Gerenciamento de Energia (Runtime PM / System PM)

- **Power-on / Runtime Resume:** Habilita `vpu_clk` → executa `reset_hw()` → retoma o escalonador (`mvx_sched_resume`).
- **Power-off / Runtime Suspend:** Suspende o escalonador (`mvx_sched_suspend`, aguardando sessões drenarem) → desabilita `vpu_clk`.
- A propriedade `clk,pm-runtime,no-sleep` no DT do K1 sinaliza que o clock não deve sofrer desligamento agressivo enquanto houver estado de escalonador/sessão ativo.
- **Papel do firmware:** O firmware **não controla** clock, tensão, reset ou domínio de energia. Ele apenas suporta economia interna via instrução `wait` (halt da CPU RASC até interrupção) e controle de *clock gating* interno (opcionalmente desabilitável por `MVE_OPT_DISABLE_FEATURE_SLEEP`).

---

## 4. Tradução de Endereços Físicos DMA no SpacemiT K1 (`[P]`)

No mapa físico do SoC SpacemiT K1, a CPU RISC-V enxerga uma janela de **IO** entre `0x00_8000_0000` e `0x00_FFFF_FFFF` (2 GiB), dividindo a DRAM em duas regiões no espaço físico da CPU:
- DRAM baixa: `0x00_0000_0000` – `0x00_7FFF_FFFF` (primeiros 2 GiB)
- Janela de IO (CPU): `0x00_8000_0000` – `0x00_FFFF_FFFF`
- DRAM alta (CPU): `0x01_0000_0000` – `0x04_7FFF_FFFF` (acima dos 2 GiB)

A VPU, porém, enxerga a DRAM de forma **contígua** no seu barramento AXI (sem o buraco de IO de 2 GiB), a partir de `0x00_0000_0000` até `0x03_FFFF_FFFF`:

```text
Espaço Físico da CPU (RISC-V)              Espaço Físico Visto pela VPU (AXI / PTE)
─────────────────────────────              ────────────────────────────────────────
0x04_8000_0000 ┌───────────┐               0x04_0000_0000 ┌───────────┐
               │           │                              │           │
               │ DRAM Alta │  ◄── (- 0x8000_0000) ──►     │ DRAM Alta │
               │           │                              │ (contígua)│
0x01_0000_0000 ├───────────┤               0x00_8000_0000 ├───────────┤
               │    IO     │  (invisível à VPU)           │           │
0x00_8000_0000 ├───────────┤                              │ DRAM Baixa│
               │ DRAM Baixa│  ◄── (identidade) ────►      │           │
0x00_0000_0000 └───────────┘               0x00_0000_0000 └───────────┘
```

### Fórmulas Normativas de Conversão (Lado Host)

- **CPU → VPU (para preencher PTEs da MMU e `MMU_CTRL`):**
  - Se `pa_cpu >= 0x1_0000_0000`: `pa_vpu = pa_cpu - 0x8000_0000`
  - Caso contrário: `pa_vpu = pa_cpu`
- **VPU → CPU (para converter um PA extraído de um PTE de volta a endereço de kernel):**
  - Se `pa_vpu >= 0x8000_0000`: `pa_cpu = pa_vpu + 0x8000_0000`
  - Caso contrário: `pa_cpu = pa_vpu`

> **Impacto no firmware:** Esta tradução é 100% transparente ao firmware RASC, pois o firmware opera exclusivamente sobre endereços virtuais de 32 bits (`mvx_mmu_va`) traduzidos pela MMU. Entretanto, qualquer driver host ou ferramenta de bancada no K1 que construa tabelas de páginas para a VPU **deve** aplicar `phys_cpu2vpu` a todos os endereços físicos (tabelas L1, tabelas L2 e páginas de dados/código), sob pena de a VPU acessar memória inválida ou gerar `MMU ABORT`.

---

*Licença: [CC-BY-SA-4.0](LICENSE) — Creative Commons Attribution-ShareAlike 4.0 International.*
