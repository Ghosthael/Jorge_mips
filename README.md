# Processador MIPS Single-Cycle em Verilog — IceSugar Nano

**EEA-25 – Sistemas Digitais Programáveis** · ITA / IEE
Prof. Dr. Harlei Miguel Arruda Leite

**Autores:** Ricardo Cardoso · Luiz Satoshi

---

## 1. Visão geral

Este projeto implementa, em **Verilog**, o processador **MIPS de ciclo único** descrito no Capítulo 7 de *Digital Design and Computer Architecture* (D. Harris & S. Harris). O livro apresenta o código em SystemVerilog e VHDL; aqui ele é **adaptado para Verilog**, validado em simulação com o **testbench da seção 7.6.3** e, por fim, sintetizado na FPGA **IceSugar Nano** (Lattice iCE40).

Em um processador de ciclo único, **cada instrução é buscada, decodificada e executada em um único ciclo de clock**. O período do clock fica limitado pelo caminho crítico mais longo (tipicamente o `lw`: memória de instruções → banco de registradores → ALU → memória de dados → banco de registradores).

---

## 2. Conjunto de instruções suportado

Subconjunto da ISA MIPS tratado no Capítulo 7:

| Instrução | Tipo | Opcode | Funct | Operação |
|-----------|------|--------|-------|----------|
| `add`  | R | `000000` | `100000` | rd ← rs + rt |
| `sub`  | R | `000000` | `100010` | rd ← rs − rt |
| `and`  | R | `000000` | `100100` | rd ← rs & rt |
| `or`   | R | `000000` | `100101` | rd ← rs \| rt |
| `slt`  | R | `000000` | `101010` | rd ← (rs < rt) ? 1 : 0 |
| `lw`   | I | `100011` | —        | rt ← Mem[rs + SignImm] |
| `sw`   | I | `101011` | —        | Mem[rs + SignImm] ← rt |
| `beq`  | I | `000100` | —        | se rs == rt: PC ← PC + 4 + (SignImm << 2) |
| `addi` | I | `001000` | —        | rt ← rs + SignImm |
| `j`    | J | `000010` | —        | PC ← {PC+4[31:28], addr, 00} |

> As tabelas de verdade do decodificador principal e do decodificador da ALU (sinais `RegWrite`, `RegDst`, `ALUSrc`, `Branch`, `MemWrite`, `MemtoReg`, `Jump`, `ALUOp`, `ALUControl`) **serão preenchidas pela dupla** em `docs/controle.md` durante a implementação.

---

## 3. Arquitetura

O projeto segue a divisão do livro entre **controle** e **datapath**:

```
top (FPGA)                      ← só para síntese: clock, reset, LEDs/pinos
└── top (simulação)             ← usado pelo testbench
    ├── mips
    │   ├── controller
    │   │   ├── maindec         ← decodifica opcode → sinais de controle
    │   │   └── aludec          ← decodifica funct + ALUOp → ALUControl
    │   └── datapath
    │       ├── flopr           ← registrador do PC (reset síncrono/assíncrono)
    │       ├── adder           ← PC + 4 e cálculo do alvo de desvio
    │       ├── sl2             ← deslocamento de 2 bits à esquerda
    │       ├── signext         ← extensão de sinal 16 → 32 bits
    │       ├── mux2            ← multiplexadores parametrizados
    │       ├── regfile         ← 32 × 32 bits, 2 leituras / 1 escrita
    │       └── alu             ← add, sub, and, or, slt + flag Zero
    ├── imem                    ← memória de instruções (carregada de arquivo)
    └── dmem                    ← memória de dados
```

### Responsabilidade de cada bloco

- **controller**: recebe `op[5:0]` e `funct[5:0]` e gera todos os sinais de controle. É lógica **puramente combinacional**.
- **datapath**: contém o PC, o banco de registradores, a ALU e os multiplexadores que escolhem os operandos e o próximo PC.
- **imem / dmem**: separadas (arquitetura Harvard), o que permite buscar a instrução e acessar dados no mesmo ciclo.

---

## 4. Adaptação SystemVerilog → Verilog

Principais pontos de tradução do código do livro:

| SystemVerilog (livro) | Verilog (este projeto) | Observação |
|---|---|---|
| `logic` | `wire` ou `reg` | `reg` quando o sinal é atribuído dentro de `always` |
| `always_ff @(posedge clk)` | `always @(posedge clk)` | usar atribuição não-bloqueante `<=` |
| `always_comb` | `always @(*)` | usar `=`; atribuir **todos** os sinais em **todos** os caminhos para evitar latches |
| `logic [31:0] RAM[63:0]` | `reg [31:0] RAM [0:63]` | declaração de memória |
| `$readmemh` | `$readmemh` | igual; verificar o caminho do arquivo na simulação e na síntese |
| `case` com `default` | `case` com `default` | o `default` é essencial em lógica combinacional |

Convenções adotadas no código:
- Um módulo por arquivo, com o nome do arquivo igual ao do módulo.
- Cabeçalho de comentário em cada módulo descrevendo entradas, saídas e função.
- `` `default_nettype none `` no início dos arquivos, para que fios não declarados gerem erro em vez de serem criados implicitamente.

---

## 5. Estrutura do repositório

```
.
├── README.md
├── rtl/                 # código Verilog sintetizável
│   ├── top.v
│   ├── mips.v
│   ├── controller.v
│   ├── maindec.v
│   ├── aludec.v
│   ├── datapath.v
│   ├── regfile.v
│   ├── alu.v
│   ├── adder.v
│   ├── sl2.v
│   ├── signext.v
│   ├── flopr.v
│   ├── mux2.v
│   ├── imem.v
│   └── dmem.v
├── fpga/
│   ├── top_fpga.v       # wrapper com os pinos da placa
│   ├── icesugar_nano.pcf
│   └── Makefile
├── sim/
│   ├── testbench.v      # adaptado da seção 7.6.3
│   └── memfile.dat      # programa de teste do livro (hex)
└── docs/
    ├── controle.md      # tabelas de verdade do controle
    └── diagramas/
```

---

## 6. Validação por simulação

### O que o testbench faz

O testbench da seção 7.6.3:
1. Instancia o `top` com `clk` e `reset`.
2. Gera reset no início e um clock periódico.
3. A memória de instruções carrega `memfile.dat`, um programa de teste que exercita **todas** as instruções suportadas.
4. A cada borda de clock, observa as escritas na memória de dados (`memwrite`, `dataadr`, `writedata`).
5. **Sucesso:** o programa escreve o valor **7** no endereço **84**. Qualquer outra escrita é considerada falha.

### Como rodar (Icarus Verilog + GTKWave)

> Ajustar se o laboratório usar outra ferramenta de simulação.

```bash
cd sim
iverilog -g2005 -o mips_tb.vvp testbench.v ../rtl/*.v
vvp mips_tb.vvp
gtkwave mips_tb.vcd     # se o testbench gerar dump de ondas
```

Resultado esperado no terminal: a mensagem de sucesso do testbench.

### Plano de verificação incremental

Antes do teste completo, testar os blocos isoladamente:
- [ ] `alu` — todas as operações + flag `Zero`
- [ ] `regfile` — leitura de `$0` sempre retorna 0; escrita na borda de subida
- [ ] `signext`, `sl2`, `mux2`, `adder`
- [ ] `maindec` / `aludec` — conferir cada linha da tabela de verdade
- [ ] Processador completo com `memfile.dat`

---

## 7. Síntese na IceSugar Nano

A IceSugar Nano usa um **iCE40LP1K** (encapsulamento CM36). Fluxo open-source:

```bash
yosys -p "synth_ice40 -top top_fpga -json mips.json" fpga/top_fpga.v rtl/*.v
nextpnr-ice40 --lp1k --package cm36 --json mips.json --pcf fpga/icesugar_nano.pcf --asc mips.asc
icepack mips.asc mips.bin
icesprog mips.bin       # gravação via iCELink (ou copiar o .bin para o drive USB da placa)
```

> Confirmar os parâmetros, a pinagem do `.pcf` e o método de gravação com o material dos laboratórios.

### Pontos de atenção (riscos técnicos)

1. **Recursos limitados.** O LP1K tem cerca de 1280 células lógicas. Um banco de 32 registradores × 32 bits implementado em flip-flops usa ~1024 FFs, quase o chip inteiro, antes de contar ALU, PC e memórias. Precisaremos analisar o relatório de utilização do `nextpnr` e avaliar alternativas (por exemplo, mapear memórias em **block RAM** e entender o que isso exige em termos de leitura síncrona × assíncrona).
2. **Leitura assíncrona de memória.** O modelo de ciclo único do livro assume leitura combinacional da imem, da dmem e do regfile. As block RAMs do iCE40 têm **leitura registrada**, então a inferência pode não acontecer como esperado.
3. **`$readmemh` na síntese.** Verificar se o Yosys inicializa corretamente a memória de instruções a partir de `memfile.dat`.
4. **Observabilidade na placa.** Em hardware não há `$display`, então é preciso definir como mostrar que o programa funcionou (por exemplo, acender um LED quando ocorrer a escrita de 7 no endereço 84, ou exibir bits de um registrador em LEDs/PMOD).
5. **Clock.** Verificar a frequência do clock disponível na placa e se o caminho crítico cabe no período (relatório de timing do `nextpnr`). Se necessário, usar um divisor de clock ou clock lento para visualização.

---

## 8. Divisão de tarefas

| Tarefa | Responsável | Status |
|--------|-------------|--------|
| Blocos básicos (`adder`, `sl2`, `signext`, `mux2`, `flopr`) | | ⬜ |
| `alu` + teste unitário | | ⬜ |
| `regfile` + teste unitário | | ⬜ |
| `maindec` / `aludec` + tabelas em `docs/controle.md` | | ⬜ |
| `datapath` e `mips` (integração) | | ⬜ |
| `imem` / `dmem` + `memfile.dat` | | ⬜ |
| Adaptação do testbench (7.6.3) | | ⬜ |
| Wrapper da FPGA + `.pcf` + Makefile | | ⬜ |
| Análise de recursos e timing | | ⬜ |
| Documentação final e instruções de uso | | ⬜ |

---

## 9. Critérios de avaliação → como atendemos

| Critério | Evidência no projeto |
|----------|---------------------|
| Datapath funcional | `rtl/datapath.v` + testes unitários dos blocos |
| Controle correto | `rtl/controller.v` + `docs/controle.md` |
| Validação funcional | `sim/testbench.v` passando com `memfile.dat` |
| Síntese na IceSugar Nano | Fluxo da seção 7 + `mips.bin` gravado e demonstrado |
| Organização e documentação | Estrutura de pastas, código comentado, este README |

---

## 10. Entrega

- **Classroom:** código Verilog completo + instruções de uso (seções 6 e 7 deste README).
- **Dia do exame:** demonstração do processador funcionando na placa.

---

## Referências

- D. M. Harris & S. L. Harris, *Digital Design and Computer Architecture*, Cap. 7 (seções 7.3 e 7.6).
- Enunciado do Projeto Verilog — EEA-25, ITA/IEE.
