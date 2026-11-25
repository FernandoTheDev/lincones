# Progresso do Desenvolvimento — LinConEs Nativo

## Estado Atual
- **Fase atual:** Fase 0 (Investigação) → Fase 1 (Fundação)
- **Compilador:** CGD v0.10.0 (`~/cgd/cgd`, `--gc=cgd`)
- **Diretório raiz:** `~/lincones`

## Concluído
- [x] Inspeção completa do ambiente e `~/cgd` (docs, stdlib, runtime, FFI, GC híbrido, projetos).
- [x] Validação experimental de `CGD-GC`, `--relatorio-memoria`, zero `libgc`, `@pilha`, `@nativo`, `Bytes`, `SQLite`, `PosixIO` (`open`/`fdopen`/`fseeko`/`fgetc_unlocked`/`fputc_unlocked`/`fflush_unlocked`/`fdatasync`/`flock`/`ftruncate`) e `LibPQ`.
- [x] Clone isolado das referências oficiais (`LinConEs`, `lincones-sqlite`, `lincones-mysql`, `lincones-postgresql`) e análise normativa em andamento.
- [x] Inicialização oficial do projeto `~/lincones` via `cgd iniciar lincones` e repositório git local.
- [x] Criação de `documentacao/INVESTIGACAO_CGD.md`.

## Próximas Ações
1. Concluir `documentacao/CONFORMIDADE_LINCONES.md` com base no estudo dos 4 repositórios oficiais.
2. Implementar Fase 1 (Fundação: diagnósticos, tipos, catálogo, testes base) e validar `cgd testar`.
3. Implementar Fase 2 (Frontend completo: Lexador, AST, Parser, Análise Semântica).
4. Implementar Fase 3 (Motor Universal: Tradutores SQLite, PostgreSQL, MySQL + Conectores).
5. Implementar Fases 4–6 (SGBD Embarcado: Formato Binário, Pager, Buffer Pool, Slotted Pages, B+ Tree, Catálogo, WAL, Transações, Concorrência).
6. Implementar Fases 7–9 (Executor Físico, Planejador/Otimizador, API Pública, CLI, REPL).
7. Implementar Fases 10–12 (Fuzzing, Testes Diferenciais com SQLite, Falha/Recuperação, Sanitizers, Benchmarks, Relatório Final).
