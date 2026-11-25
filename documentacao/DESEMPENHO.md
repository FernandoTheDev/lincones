# Desempenho e Telemetria do CGD-GC

## 1. Metodologia

- **Hardware / SO:** Linux x86_64.
- **Compilador:** `~/cgd/cgd` (`CGD v0.10.0`, flags `--gc=cgd -o`).
- **Benchmark reproduzível:** `desempenho/benchmark_lincones_vs_sqlite.delegua` comparando `MotorProprio` (`:memoria:`) vs `ConectorSQLite` (`:memory:` sobre `libsqlite3.so.3.51.2`) em condições equivalentes, com os mesmos comandos LinConEs e parâmetros vinculados.

## 2. Resultados Medidos (Escala: 1.000 registros)

| Operação | Motor Próprio (Antes) | Motor Próprio (Após Otimização) | SQLite C (Referência) | Razão (Próprio / SQLite) |
| :--- | :--- | :--- | :--- | :--- |
| Inserção em transação (1.000 linhas) | 4023.34 ms | 4160.56 ms | 67.25 ms | 61.9x |
| 200 buscas por PK (B+ Tree) | 25.82 ms | **23.40 ms** | 12.09 ms | **1.93x** |
| Varredura + Agregação (`CONTAR`, `SOMA`) | 4.56 ms | 4.65 ms | 0.18 ms | 25.8x |
| Pico de Heap CGD-GC (`heapBytes`) | 59.38 MiB | **33.75 MiB (-43%)** | N/A | — |
| Índices B+ Tree selecionados pelo otimizador | 200 / 200 | **200 / 200** | 200 / 200 | 100% |

## 3. Otimizações Automáticas do CGD-GC Confirmadas no Binário Final (`compilacao/lincones`)

Relatório completo salvo em `documentacao/relatorio_memoria_cgd.json` (`--relatorio-memoria-json`):

- **Funções especializadas automaticamente (ANS):** `30` (incluindo `atualizarByteCrc32`, `potenciaDez`, `triE`, `triOu`, `triNao`, constantes de página e WAL).
- **Chamadas diretas unboxed (ANS):** `53`.
- **Objetos promovidos para stack local:** `2` (`parser:AnalisadorSintatico`, `leitor:LeitorJSON`).
- **Coleções com liberação determinística em O(1):** `9` (`posRef`, `nomesVistos`, `vals`, `chavesGrupo`, `grupos`, `vistas`, `chavesOrd`, `descFlags`, `larguras`).
- **Chamadas de método desvirtualizadas:** `403`.
- **Dependência de `libgc`:** **Nenhuma** (confirmado via `ldd compilacao/lincones`).

## 4. Gargalos Reais Identificados

1. `ArmazenamentoTabela.inserirRegistro` percorre a cadeia de páginas desde `paginaRaizDados` e recalcula CRC32 de 4096 bytes por inserção — manter `ultimaPaginaDados` na `DefinicaoTabela` e CRC32 diferido até flush reduzirá drasticamente o tempo de inserção em lote.
2. Conversão e re-parsing de consultas repetidas no benchmark (ausência de cache de planos preparados no `MotorProprio`).
