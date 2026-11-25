# Transações, Write-Ahead Logging (WAL) e Recuperação

## 1. Modelo Transacional (ACID)

- **Atomicidade:** antes de iniciar uma transação explícita, o `GerenciadorWal` captura snapshots de desfazer (*undo*) das páginas existentes. Em caso de `REVERTER` ou exceção, todas as páginas e o catálogo retornam exatamente ao estado inicial.
- **Pontos de Salvamento (`Savepoints`):** `PONTO DE SALVAMENTO <nome>`, `REVERTER PARA <nome>` e `LIBERAR PONTO <nome>` restauram o estado exato de páginas e catálogo no ponto solicitado sem desfazer operações anteriores da mesma transação.
- **Isolamento:** modelo de **múltiplos leitores e escritor único** (`ControleLocks`) em memória, combinado com trava exclusiva `flock(LOCK_EX | LOCK_NB)` no descritor POSIX do arquivo `.lce` para rejeitar aberturas multiprocesso concorrentes incompatíveis.
- **Durabilidade:** o protocolo Write-Ahead Logging grava todas as páginas alteradas + quadro `COMMIT` com CRC32 individual no arquivo `.lce-wal` e executa `fdatasync(walFd)` **antes** de descarregar páginas no arquivo principal `.lce`.

## 2. Protocolo de Confirmação e Checkpoint

1. `INICIAR TRANSACAO` adquire trava de escrita e captura snapshot base.
2. Modificações ocorrem no Buffer Pool.
3. `CONFIRMAR`:
   - Serializa catálogo atualizado.
   - Grava todas as páginas da transação + quadro `COMMIT` no `.lce-wal` com CRC32 por quadro.
   - Executa `fflush_unlocked` + `fdatasync` no `.lce-wal`.
   - Descarrega páginas sujas no `.lce` e executa `fdatasync` no `.lce` (checkpoint).
   - Remove `.lce-wal`.

## 3. Recuperação Idempotente

Na abertura do banco (ou via `lincones recuperar <banco.lce>`):

- Se `.lce-wal` existe, valida o cabeçalho WAL (`WAL1` + CRC32).
- Lê quadros sequencialmente validando o CRC32 de cada quadro. Ao encontrar EOF parcial ou checksum inválido, interrompe a leitura e descarta transações incompletas (sem quadro `COMMIT`).
- Aplica exclusivamente transações com quadro `COMMIT` íntegro ao arquivo `.lce`, executa checkpoint durável e remove `.lce-wal`.
- Validado experimentalmente em `testes/transacoes_wal_teste.delegua` com simulação real de crash após WAL com `COMMIT` (recuperado) e após WAL sem `COMMIT` (descartado), além de idempotência em recuperações consecutivas.
