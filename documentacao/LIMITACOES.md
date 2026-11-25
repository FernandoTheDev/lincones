# Limitações Conhecidas e Histórico de Resoluções — LinConEs Nativo

Este documento mantém registro transparente do que foi **resolvido** durante o ciclo de engenharia (incluindo correções estruturais no CGD) e das **limitações reais remanescentes** no estado atual do projeto.

## 1. Limitações Anteriores Resolvidas

| Item Anterior | Status Atual | Como foi Resolvido |
|---|---|---|
| Corrupção de strings de 1 byte no `CGD-GC` sob pressão mista | **RESOLVIDO** | Corrigidos os 6 pontos no runtime `delegua-runtime` que ignoravam `DELEGUA_T_TEXT` com `len < 2`; fortalecido `HeapVerifier` para minor+major; validado em 6/6 perfis, ASan+UBSan e fuzzing 10×. |
| Conflito de cabeçalhos `<stdio.h>` e falta de `fclose` no FFI do CGD | **RESOLVIDO** | Introduzido `cponteiro` (`void *`), metadado `declarado` e detecção automática de símbolos da libc no CGD; `posix_io.delegua`, `gerenciador_paginas.delegua`, `wal.delegua` e `copiarPara` modernizados com `fclose` após `fdopen`. |
| Linkagem de bibliotecas por soname versionado (`libmariadb.so.3`) | **RESOLVIDO** | Adicionado suporte a `:libfoo.so.N` e caminhos absolutos no FFI/driver do CGD; `ConectorMySQL` validado ao vivo contra `11.8.8-MariaDB`. |
| Validação ao vivo de conectores externos (MariaDB e PostgreSQL) | **RESOLVIDO** | SQLite (`3.51.2`), MariaDB (`11.8.8-MariaDB` isolado via `:libmariadb.so.3`) e PostgreSQL (`17` isolado via `libpq 180000`) validados ao vivo de ponta a ponta com `CGD_GC_PROFILE=verify`. |
| Limite de ~4000 bytes por registro (`SlottedPage`) | **RESOLVIDO** | Implementadas páginas de overflow encadeadas (`tipo = 7`, 4076 B/página + CRC32) e prefixo `u32` para textos; validado até **1 MiB** com persistência, atualização, exclusão, rollback e reaproveitamento de páginas livres. |
| Ordenação (`ORDENAR POR`) limitada à memória RAM | **RESOLVIDO** | Implementado `OrdenadorExterno` (`ordenacao_externa.delegua`) com orçamento configurável (`orcamentoOrdenacaoBytes`), arquivos temporários exclusivos (`O_EXCL \| 0600`), k-way merge estável e limpeza garantida em `tente/finalmente`. |
| Ausência de CTEs, subconsultas correlacionadas e funções de janela | **RESOLVIDO** | Implementados `COM ... COMO`, `EXISTE`, `NAO EXISTE`, `EM (SELECIONAR ...)`, subconsultas escalares e `SOBRE (PARTICIONAR POR ... ORDENAR POR ...)` com `NUMERO_LINHA`, `CLASSIFICACAO`, `CLASSIFICACAO_DENSA`, `ANTERIOR`, `PROXIMO` e agregações de janela, com validação diferencial contra SQLite. |

## 2. Limitações Reais Remanescentes

1. **CTEs Recursivas (`COM RECURSIVO`)**: CTEs não-recursivas (simples e encadeadas) são suportadas tanto na tradução quanto no `MotorProprio`; recursão de ponto fixo (`UNION ALL` recursivo) não foi implementada nesta versão.
2. **Concorrência multi-processo / MVCC**: o `MotorProprio` implementa travamento cooperativo com detecção de conflitos em nível de tabela/banco em processo único, não oferecendo isolamento `SNAPSHOT` multi-escritor simultâneo entre processos distintos.
3. **Leitura célula-a-célula no `ConectorMySQL` via FFI**: `libmariadb` expõe linhas via `MYSQL_ROW` (`char **`), cuja desreferenciação direta de ponteiro-para-ponteiro não possui primitiva segura em Delégua sem código C manual. O `ConectorMySQL` executa SQL/DDL/DML/transações/CTEs/janelas ao vivo e retorna contagens reais (`linhasAfetadas`, `numLinhas`, `numColunas`); o `ConectorPostgreSQL` (`libpq`) e o `ConectorSQLite` retornam todas as células completas linha a linha porque `PQgetvalue` e `cgd:dados/sqlite` devolvem `const char *` / valores diretos.
4. **Distinção MariaDB vs MySQL**: o conector externo foi validado ao vivo contra servidor **MariaDB 11.8.8** usando `libmariadb.so.3`. O dialeto SQL gerado é compatível com MySQL 8+ e MariaDB 10.2+, mas não houve execução ao vivo contra binário Oracle MySQL Server neste host.
5. **Desempenho de inserção em lote vs SQLite em C**: o `MotorProprio` é escrito 100% em Delégua (com `CGD-GC`); inserções individuais de 1000 linhas em memória levam ~4,3 s (vs ~65 ms no SQLite em C), enquanto buscas indexadas por B+ Tree (200 consultas) levam ~22 ms (vs ~10 ms no SQLite).
