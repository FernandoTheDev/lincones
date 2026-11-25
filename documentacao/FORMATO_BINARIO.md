# Formato Binário do Arquivo `.lce` (Versão 1)

Todo arquivo `.lce` é organizado em **páginas de tamanho fixo de 4096 bytes**, em ordem de bytes **little-endian** explícita, sem gravação crua de estruturas de memória.

## 1. Página 0 — Cabeçalho do Banco

| Deslocamento | Tamanho | Tipo | Campo |
| :--- | :--- | :--- | :--- |
| `0` | 4 B | `u32 LE` | Assinatura mágica `LCE1` (`0x3145434C` = `826622796`) |
| `4` | 4 B | `u32 LE` | Versão do formato (`1`) |
| `8` | 4 B | `u32 LE` | Tamanho da página (`4096`) |
| `12` | 4 B | `u32 LE` | Total de páginas alocadas no arquivo |
| `16` | 4 B | `u32 LE` | Cabeça da lista de páginas livres (`0` = vazia) |
| `20` | 4 B | `u32 LE` | ID da página raiz do catálogo (`1`) |
| `24` | 4 B | `u32 LE` | Versão do esquema |
| `28` | 4 B | `u32 LE` | Próximo `txId` |
| `32` | 4 B | `u32 LE` | Flags de integridade (`0`) |
| `36` | 4 B | `u32 LE` | CRC32 IEEE 802.3 dos bytes `[0..35]` |

## 2. Cabeçalho Comum das Páginas (`idPagina >= 1`)

| Deslocamento | Tamanho | Tipo | Campo |
| :--- | :--- | :--- | :--- |
| `0` | 1 B | `u8` | Tipo da página: `2=Dados`, `3=BTreeFolha`, `4=BTreeInterna`, `5=Livre`, `6=Catalogo`, `7=Overflow` |
| `1` | 1 B | `u8` | Flags reservadas |
| `2` | 2 B | `u16 LE` | Quantidade de slots ou entradas |
| `4` | 2 B | `u16 LE` | Ponteiro de espaço livre inferior (`livreInicio`) |
| `6` | 2 B | `u16 LE` | Ponteiro de espaço livre superior (`livreFim`) |
| `8` | 4 B | `u32 LE` | Próxima página (encadeamento de dados, cadeia de folhas B+ Tree, ou `filho[0]` em nó interno) |
| `12` | 4 B | `u32 LE` | CRC32 IEEE 802.3 calculado sobre `[0..11]` concatenado com `[16..4095]` |

## 3. Slotted Pages de Dados (`tipo = 2`)

- Diretório de slots cresce a partir do byte `16`: cada slot ocupa 4 bytes (`offset: u16 LE`, `tamanho: u16 LE`).
- Payloads de registros crescem do byte `4096` para baixo.
- Slots excluídos têm `offset = 0` e `tamanho = 0`, preservando os `RID (idPagina, idSlot)` dos demais registros durante compactações.
- Cada tupla serializada contém `numColunas (u16 LE)`, bitmap de nulidade (`ceil(numColunas / 8)` bytes) e valores tipados em formato explícito.

## 4. Páginas de Índice B+ Tree (`tipo = 3` e `tipo = 4`)

- **Folha (`tipo = 3`):** `[8..11]` aponta para a próxima folha em ordem crescente; entradas sequenciais `[tamChave: u16 LE, chaveSerializada, ridCompacto: u64 LE]`.
- **Interna (`tipo = 4`):** `[8..11]` armazena `filho[0]`; entradas sequenciais `[tamChave: u16 LE, chaveSerializada, filhoDireito: u32 LE]`.

## 5. Arquivo WAL (`.lce-wal`)

- Cabeçalho de 16 bytes: assinatura `WAL1` (`0x314C4157`), versão `1`, tamanho de página `4096`, CRC32 de `[0..11]`.
- Sequência de quadros de 4116 bytes: `txId (u32)`, `tipoQuadro (u32: 1=Pagina, 2=Commit)`, `idPagina (u32)`, `totalPaginas (u32)`, `crc32Quadro (u32)` e imagem integral de `4096` bytes da página.
