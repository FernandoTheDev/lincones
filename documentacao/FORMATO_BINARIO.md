# Formato Binário em Disco (`.lce` e `.lce-wal`) — LinConEs Nativo

## 1. Organização Geral das Páginas

- **Tamanho fixo de página**: `4096 bytes` (4 KiB), endianness **little-endian**.
- **Integridade**: toda página contém checksum **CRC32** nos últimos 4 bytes (`offset 4092..4095`), validado na leitura.
- **Cabeçalho comum de página**:
  - `offset 0`: `u8 tipoPagina`
    - `1` = Cabeçalho do banco (página 1)
    - `2` = Página de dados (`SlottedPage`)
    - `3` = Nó interno de Árvore B+
    - `4` = Nó folha de Árvore B+
    - `5` = Página livre (lista encadeada de páginas reutilizáveis)
    - `6` = Catálogo do sistema
    - `7` = **Página de Overflow** (registros > 3900 bytes)

## 2. Cabeçalho do Banco (Página 1)

- `0..7`: Assinatura mágica `"LINCONES"` (8 bytes)
- `8..11`: Versão do formato (`u32`, atual `1`)
- `12..15`: Tamanho da página (`u32 = 4096`)
- `16..19`: Total de páginas alocadas (`u32`)
- `20..23`: Página raiz do catálogo (`u32`)
- `24..27`: Cabeça da lista de páginas livres (`u32`, `0` = vazia)
- `28..31`: Contador de geração / schema version (`u32`)

## 3. Páginas de Dados (`SlottedPage`, `tipo = 2`) e Páginas de Overflow (`tipo = 7`)

- Registros de até **3900 bytes** são armazenados integralmente *inline* dentro da `SlottedPage`.
- Registros maiores que **3900 bytes** (testados até **1 MiB**, com suporte teórico até 16 MiB por campo de texto via prefixo `u32` na serialização de tuplas) utilizam **Páginas de Overflow encadeadas (`tipo = 7`)**:
  - **Envelope inline na `SlottedPage` (12 bytes)**:
    - `0..1`: Marcador `0xFFFF` (`u16`)
    - `2..3`: Assinatura `'O'` (`0x4F`), `'V'` (`0x56`)
    - `4..7`: `u32 tamanhoTotal` do registro serializado
    - `8..11`: `u32 primeiraPaginaOverflow`
  - **Layout da Página de Overflow (`tipo = 7`, 4096 bytes)**:
    - `0`: `u8 tipo = 7`
    - `1..3`: Reservado (`0`)
    - `4..7`: `u32 proximaPagina` (`0` indica última página da cadeia)
    - `8..11`: `u32 bytesUsados` (`0..4076`)
    - `12..15`: `u32 indiceSequencia` (0, 1, 2, ... para detecção de ciclos/corrupção)
    - `16..4091`: Carga útil (`4076 bytes` máximos por página)
    - `4092..4095`: `u32 crc32`
  - **Transacionalidade e reciclagem**: exclusões, atualizações (overflow → inline ou overflow → overflow) e `REVERTER` liberam todas as páginas da cadeia de overflow devolvendo-as à lista de páginas livres (`tipo = 5`), sendo reutilizadas nas próximas alocações.

## 4. Write-Ahead Log (`.lce-wal`)

- Arquivo complementar `<banco>.lce-wal` contendo quadros sequenciais com cabeçalho (número da página, tamanho, número da transação, flag de commit, CRC32) e imagem completa da página de 4 KiB.
- Suporta `INICIAR TRANSACAO`, `CONFIRMAR`, `REVERTER`, `PONTO DE SALVAMENTO`, `REVERTER PARA` e recuperação automática após falha (redo de transações confirmadas, descarte de transações incompletas).
