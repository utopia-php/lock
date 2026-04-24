# Changelog

## 0.1.0

Initial release.

- `Utopia\Lock\Lock` interface with `acquire`, `tryAcquire`, `release`, `withLock`.
- `Utopia\Lock\Lock\Mutex` — Swoole coroutine mutex backed by `Swoole\Coroutine\Channel(1)`.
- `Utopia\Lock\Lock\Semaphore` — Swoole coroutine semaphore with a configurable permit count.
- `Utopia\Lock\Lock\File` — `flock()` based file lock (`LOCK_EX` or `LOCK_SH`), cooperates with Swoole's runtime hooks.
- `Utopia\Lock\Lock\Distributed` — Redis `SET NX EX` lock with Lua-atomic release and refresh, fractional-second timeouts, optional logger.
- `Utopia\Lock\Exception` base exception and `Utopia\Lock\Exception\Contention` thrown when `withLock` times out.
