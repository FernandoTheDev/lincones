# Desempenho, Telemetria e Validação de Memória — LinConEs Nativo

## 1. Benchmark Comparativo Pós-Correção (`MotorProprio` vs `SQLite C`)

Executado via [benchmark_lincones_vs_sqlite.delegua](file:///home/fernandodev/lincones/desempenho/benchmark_lincones_vs_sqlite.delegua) compilado com `-o` (otimização máxima C99 do CGD) em modo `:memoria:` equivalente (1000 registros, 200 buscas pontuais por chave primária via Árvore B+, 1 varredura com agregação):

| Configuração | Inserção 1000 regs | 200 PK B+ Tree | Scan + Agregação | Ciclos GC | Liberados Determinísticos |
| :--- | ---: | ---: | ---: | ---: | ---: |
| **LinConEs `MotorProprio` (`CGD-GC` padrão)** | 4340,29 ms | 22,45 ms | 4,83 ms | 11 | 4 |
| **LinConEs `MotorProprio` (`Boehm` ref.)** | 4310,66 ms | 21,87 ms | 4,14 ms | 91 | 0 |
| **SQLite C (`3.51.2` via `cgd:dados/sqlite`)** | 67,17 ms | 9,76 ms | 0,11 ms | — | — |

**Observações:**
- O coletor padrão **`CGD-GC`** atingiu paridade de desempenho com o **`Boehm`** (< 1% de diferença), executando **8,3× menos ciclos de GC** (11 vs 91) e preservando liberações determinísticas híbridas.
- O binário de produção `compilacao/lincones` tem **zero** dependência de `libgc`, `libsqlite3`, `libpq` ou `libmariadb`.

## 2. Estresse, Fuzzing 10× e Sanitizers

| Experimento | Escopo | Resultado |
| :--- | :--- | :--- |
| Caso reproduzível `cgd_gc_fuzz_pressao.delegua` | 6 perfis (`balanced`, `low_mem`, `throughput`, `low_latency`, `stress` [117.931 ciclos], `verify` [minor+major]) × GCC `-O2` e Clang `-O2` | **PASSOU (12/12)** |
| Fuzzing intensificado 10× (`ferramentas/fuzz_intensivo_10x.delegua`) | 3 sementes × (4000 parses + 1500 ops MotorProprio) = 12.000 parses + 4.500 ops em `balanced`, `low_mem`, `verify`, `boehm` | **PASSOU (4/4)** |
| **ASan + UBSan** + `CGD_GC_PROFILE=verify` | `cgd_gc_fuzz_pressao.delegua` e `overflow_e_ordenacao_externa_teste.delegua` (registros até 1 MiB + external sort em disco) | **PASSOU (0 erros)** |
| Overflow Pages + External Sort | Persistência de registros 4001 B, 8 KiB, 64 KiB e **1 MiB** + ordenação externa forçada (orçamento 2 KiB sobre 120 linhas) | **PASSOU** |
