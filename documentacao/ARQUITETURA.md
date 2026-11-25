# Arquitetura do LinConEs Nativo

## Fluxo Principal

```text
Código LinConEs
  → Lexador (fontes/lexador/lexador.delegua)
  → Analisador Sintático (fontes/sintatico/parser.delegua)
  → AST Independente de Banco (fontes/ast/nos.delegua)
  → Analisador Semântico (fontes/semantico/analisador.delegua) sobre Catálogo Abstrato
  ├── Motor Universal SQL:
  │     ├── TradutorSQLite + ConectorSQLite (cgd:dados/sqlite)
  │     ├── TradutorPostgreSQL + ConectorPostgreSQL (libpq condicional)
  │     └── TradutorMySQL + ConectorMySQL
  └── SGBD Embarcado Próprio (MotorProprio):
        ├── Otimizador Baseado em Custo (dobramento de constantes + seleção de B+ Tree)
        ├── Executor Físico (índice, varredura, junção, filtro, agrupamento, ordenação, paginação)
        ├── Gerenciador WAL + Transações + Savepoints + ControleLocks
        ├── Árvore B+ Persistente (4096 B) + Armazenamento Slotted Pages + Catálogo Persistente
        └── Gerenciador de Páginas + Buffer Pool CLOCK + PosixIO (fdatasync/flock)
```

## Módulos em `fontes/`

- `diagnosticos/diagnostico.delegua`: posições, intervalos, níveis e coletor de diagnósticos.
- `tipos/valor.delegua`: sistema de tipos relacionais, decimal escalado em 64 bits, lógica ternária e ordenação total.
- `serializacao/crc32.delegua`: CRC32 IEEE 802.3 em aritmética escalar (especializado via ANS pelo CGD).
- `catalogo/esquema.delegua` e `catalogo/persistencia_catalogo.delegua`: catálogo abstrato e persistência binária multi-página.
- `lexador/` e `sintatico/` e `ast/` e `semantico/`: frontend único com modo estrito e modo estendido.
- `traducao/` e `conectores/`: tradutores SQLite, PostgreSQL e MySQL + conectores.
- `armazenamento/`, `paginas/`, `indices/`, `transacoes/`, `concorrencia/`: núcleo de persistência e ACID do SGBD próprio.
- `expressoes/`, `otimizacao/`, `execucao/`: avaliação de expressões, otimização e execução física.
- `api/lincones.delegua`: fachada pública consumível por outros projetos CGD.
- `terminal/formatacao.delegua`, `terminal/cli.delegua`, `principal.delegua`: CLI, REPL e saída tabular/JSON/CSV.
