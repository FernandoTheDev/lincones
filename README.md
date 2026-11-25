# LinConEs Nativo

Implementação nativa, completa e modular da **Linguagem de Consulta Estruturada (LinConEs)** em **Delégua**, compilada nativamente pelo **CGD v0.10.0** com coletor híbrido **CGD-GC** (zero dependência de `libgc`).

O projeto entrega dois produtos integrados sobre um único frontend compartilhado:

1. **Motor Universal de Consultas SQL:** análise léxica, sintática e semântica única com tradução parametrizada para **SQLite**, **PostgreSQL** e **MySQL**, além de conectores modulares (SQLite integrado via `cgd:dados/sqlite`, PostgreSQL via `libpq` condicional `--recurso postgresql`, MySQL modular).
2. **SGBD Relacional Embarcado Próprio (`.lce`):** sistema gerenciador de banco de dados relacional genuíno, escrito 100% em Delégua, sem uso de SQLite ou qualquer outro SGBD interno, com:
   - Formato binário paginado próprio de **4096 bytes** com checksum **CRC32 IEEE 802.3** por página.
   - Gerenciador de páginas via **PosixIO** (`open`/`fdopen`/`fseeko`/`fdatasync`/`flock`) com **Buffer Pool CLOCK** de orçamento configurável (manipula bancos maiores que o cache configurado).
   - **Slotted Pages** com bitmap de nulidade, registros de tamanho variável, compactação e `RID` estáveis.
   - **Índices B+ Tree persistentes** em páginas de 4096 bytes com splits, busca exata, busca por intervalo e verificador estrutural.
   - **Catálogo persistente**, chaves primárias, auto incremento, restrições `NAO NULO`, `UNICA` e integridade referencial (`CHAVE ESTRANGEIRA`).
   - **Transações ACID**, **Write-Ahead Logging (`.lce-wal`)** com recuperação idempotente após falhas e **Pontos de Salvamento** (`PONTO DE SALVAMENTO` / `REVERTER PARA`).
   - **Otimizador baseado em custo** (dobramento de constantes com lógica ternária e seleção automática de índices B+ Tree) e **Executor Físico** com junções, agrupamento, agregações, ordenação estável, `DISTINTO`, `LIMITE`, `DESLOCAMENTO` e `EXPLICAR [ANALISAR]`.

## Compilação e Uso

```bash
cd ~/lincones
~/cgd/cgd compilar --modo producao
~/cgd/cgd testar
```

### Exemplos de CLI

```bash
./compilacao/lincones versao
./compilacao/lincones traduzir --motor=postgresql -c 'CRIAR TABELA t (id INTEIRO CHAVE PRIMARIA AUTO INCREMENTO, nome TEXTO(80));'
./compilacao/lincones executar meu_banco.lce exemplos/basico.lincones
./compilacao/lincones executar meu_banco.lce -c 'SELECIONAR * DE livros;' --formato=json
./compilacao/lincones explicar meu_banco.lce -c 'SELECIONAR * DE livros ONDE id = 1;'
./compilacao/lincones verificar meu_banco.lce
./compilacao/lincones copia meu_banco.lce backup.lce
./compilacao/lincones estatisticas meu_banco.lce
./compilacao/lincones abrir meu_banco.lce   # Abre REPL interativo
```

## Documentação

- [INVESTIGACAO_CGD.md](documentacao/INVESTIGACAO_CGD.md)
- [CONFORMIDADE_LINCONES.md](documentacao/CONFORMIDADE_LINCONES.md)
- [ARQUITETURA.md](documentacao/ARQUITETURA.md)
- [FORMATO_BINARIO.md](documentacao/FORMATO_BINARIO.md)
- [TRANSACOES_E_RECUPERACAO.md](documentacao/TRANSACOES_E_RECUPERACAO.md)
- [DESEMPENHO.md](documentacao/DESEMPENHO.md)
- [LIMITACOES.md](documentacao/LIMITACOES.md)
- [RELATORIO_FINAL.md](documentacao/RELATORIO_FINAL.md)
