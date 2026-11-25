# Investigação Técnica do Compilador Geral Delégua (CGD)

## 1. Ambiente e Versão Verificados

- **Diretório do compilador:** `~/cgd`
- **Binário utilizado:** `~/cgd/cgd` (`CGD - Compilador Geral Delégua v0.10.0`)
- **Commit base observado em `~/cgd`:** `ef68db3` (branch `dev`, com árvore de trabalho contendo a implementação recente do CGD-GC híbrido, sistema de projetos e Prompt Mestre X)
- **Plataforma:** Linux x86_64
- **Compiladores C disponíveis:** `clang` (`/usr/bin/clang`), `gcc` (`/usr/bin/gcc`)
- **Bibliotecas de bancos externos no sistema:**
  - `libsqlite3.so` (`/lib64/libsqlite3.so.3.51.2`) + integração nativa no runtime via `cgd:dados/sqlite`
  - `libpq.so` (`/lib64/libpq.so.5.18`, PostgreSQL 18, confirmado `PQlibVersion() = 180000`)
  - `libmariadb.so.3` (`/lib64/libmariadb.so.3`, sem symlink de desenvolvimento `.so`) e binários `mariadbd`/`mysql`

## 2. Sistema Oficial de Projetos e Módulos

Confirmado via `~/cgd/docs/PROJETOS.md`, `~/cgd/docs/MODULOS.md` e execução real de `~/cgd/cgd iniciar lincones`:

- Manifesto oficial: `projeto.json`
- Estrutura padrão: `fontes/`, `testes/`, `compilacao/`, `recursos/`
- Comandos oficiais:
  - `cgd iniciar <nome> [--tipo aplicativo|servidor|tui|gtk|sdl|biblioteca]`
  - `cgd compilar [arquivo] [--modo desenvolvimento|producao] [--depuracao]`
  - `cgd rodar [arquivo] [-- argumentos...]`
  - `cgd testar [--filtro texto]` (descobre `*_teste.delegua` em `testes/`)
  - `cgd verificar [arquivo]`
  - `cgd formatar [caminho] [--verificar]`
  - `cgd projeto [validar|informacoes]`
- **Regra crítica de módulos e segurança (confirmada experimentalmente):**
  - `const X = importar("...")` cria variável local de módulo; funções dentro do arquivo **não** podem capturá-la (`W4001`/`E4001`: captura externa/upvalues bloqueada por segurança).
  - A forma idiomática e obrigatória para namespaces reutilizáveis em funções e anotações de tipos é a importação qualificada estilo D:
    ```delegua
    modulo lincones.lexador.lexador
    importar cgd.dados.binario como Bin
    importar lincones.diagnosticos.diagnostico como Diag
    ```
  - `SymbolModule` é um namespace estático acessível em funções e métodos sem gerar captura.

## 3. Recursos Fundamentais Validados Experimentalmente

### 3.1. Gerenciamento Híbrido de Memória e `CGD-GC`

Confirmado experimentalmente com `--gc=cgd --opcoes-memoria=padrao --relatorio-memoria` e inspeção via `ldd`:

- O executável gerado usa `CGD-GC` por padrão (`Memoria.estatisticas()["backendGC"] == "cgd-gc"`).
- **Zero dependência de `libgc` (Boehm GC)** confirmada via `ldd`.
- Flags confirmadas:
  - `--gc=cgd` (padrão) / `--gc=boehm` (referência experimental)
  - `--opcoes-memoria=padrao|nenhuma|ans,stack,det,fusao,devirt,safepoints`
  - `--relatorio-memoria`
  - `--relatorio-memoria-json=<arquivo.json>`
- Variáveis de ambiente em runtime:
  - `CGD_GC_PROFILE=balanced|low_mem|throughput|low_latency|stress|verify`
  - `CGD_GC_STATS=1`
- Otimizações disponíveis no compilador:
  - **ANS (Especialização Nativa Automática):** funções escalares puras compiladas unboxed (`i64`/`f64`/`bool`).
  - **Promoção para stack (`StackLocal`):** classes fechadas sem escape alocadas na stack.
  - **Liberação determinística (`DeterministicHeap`):** coleções locais sem escape recicladas em O(1) na TLAB.
  - **Fusão de tuplas, pré-dimensionamento de vetores e desvirtualização de chamadas.**

### 3.2. Tipos Nativos e Anotações (`@nativo`, `@pilha`, `@puro`, `@quando`)

Confirmados experimentalmente:

- `@pilha` / `@valor`: structs com semântica de valor e layout C fixo.
- `@nativo`: funções C99 sem boxing sobre tipos primitivos C (`ci8`..`ci64`, `cu8`..`cu64`, `cf32`, `cf64`, `cbool`).
- `@quando(Compilacao.temRecurso("..."))`: compilação condicional estrutural avaliada antes do carregamento de módulos e da linkagem (ideal para conectores externos opcionais).

### 3.3. I/O Binário Aleatório, Durabilidade e Bloqueio de Arquivos (POSIX FFI)

A biblioteca padrão `cgd:sistema/arquivos` e `cgd:dados/binario` (`Bytes`) oferece leitura/escrita integral de arquivos, manipulação little-endian (`lerU16LE`, `lerU32LE`, `lerU64LE`, `definirU32LE`, `definirU64LE`) e arenas (`cgd:sistema/memoria`).

Para acesso aleatório por páginas de 4096 bytes (obrigatório para manipular bancos maiores que a memória RAM), sincronização durável (`fsync`/`fdatasync`) e travas multiprocesso (`flock`), validamos experimentalmente a FFI C nativa do CGD com `libc`:

- **Limitação descoberta e documentada:** funções declaradas em `<stdio.h>` padrão C99 (`fopen`, `fclose`, `fseek`, `fputc`, `fgetc`, `fflush`) geram conflito de tipo `FILE *` vs `uint64_t` porque o runtime C99 gerado inclui `<stdio.h>`.
- **Solução idiomática e 100% Delégua (validada com sucesso):** utilizar as primitivas POSIX que não conflitam com `<stdio.h>` em C99 estrito:
  - `open`, `close`, `lseek`, `fsync`, `fdatasync`, `ftruncate`, `flock`, `unlink`, `rename`
  - `fdopen`, `fseeko`, `fgetc_unlocked`, `fputc_unlocked`, `fflush_unlocked`
- **Resultado do experimento (`teste_io_pagina.delegua`):** escrita de 4096 bytes no deslocamento 4096, `fflush_unlocked`, `fdatasync`, releitura integral e verificação byte-a-byte com sucesso (`pagina_4k_ok=verdadeiro`), compilado com `--gc=cgd -o`.

### 3.4. Conectores de Bancos Externos

- **SQLite:** nativamente integrado via `cgd:dados/sqlite` (`SQLite.abrir`, `executar`, `consultar`, `ultimoId`, `iniciarTransacao`, `confirmar`, `reverter`, `fechar`). Validado experimentalmente com sucesso.
- **PostgreSQL:** `libpq.so` validada via `@definicao(biblioteca = "pq", abi = "c")` (`PQlibVersion() = 180000`). Será integrada de forma condicional/modular para não impor dependência obrigatória ao motor embarcado.
- **MySQL/MariaDB:** `libmariadb.so.3` está presente sem symlink `.so`; o conector suportará integração nativa e validação isolada quando habilitado, mantendo compilação independente do motor próprio.

### 3.5. Concorrência e Rotinas

- `rotina` (`Tarefa<T>`) executa funções puras/isoladas sobre pool nativo de workers Linux com `eventfd`/`epoll` e `Canal`/`GrupoTarefas`.
- O runtime principal é projetado para execução controlada com estado TLS por thread e auditoria estática de efeitos (`E3401`-`E3404`).
- Para concorrência entre múltiplas conexões e processos no SGBD embarcado, combinaremos controle explícito de locks de leitura/escrita (múltiplos leitores, escritor único) em memória e `flock` POSIX no arquivo de banco/lock para proteção multiprocesso.

## 4. Decisões Arquiteturais Derivadas da Investigação

1. Usar `fontes/` e `testes/` conforme o padrão oficial do `projeto.json` do CGD.
2. Usar exclusivamente importações qualificadas (`importar ... como ...`) em todos os módulos.
3. Escrever o gerenciador de páginas sobre `PosixIO` (FFI `libc` puro em Delégua) com buffers `Bytes` de 4096 bytes e `fdatasync`/`fsync` reais.
4. Usar classes fechadas com campos tipados concretos e funções escalares nos caminhos quentes (lexador, serialização, CRC32, busca binária na B+ Tree) para maximizar ANS, desvirtualização e alocação em stack pelo CGD.
5. Manter o executável principal `lincones` 100% livre de `libgc` e sem dependências obrigatórias de bancos externos.
