# Utopia Lock

[![Build Status](https://github.com/utopia-php/lock/actions/workflows/tests.yml/badge.svg)](https://github.com/utopia-php/lock/actions/workflows/tests.yml)
[![License](https://img.shields.io/github/license/utopia-php/lock.svg)](https://github.com/utopia-php/lock/blob/main/LICENSE)

Four lock primitives behind a single interface, for PHP 8.3+ on Swoole 6.

## Installation

```bash
composer require utopia-php/lock
```

## When to use which

| Primitive     | Scope                                 | Backing                              | Use when                                                    |
| ------------- | ------------------------------------- | ------------------------------------ | ----------------------------------------------------------- |
| `Mutex`       | Single worker, coroutine-scoped       | `Swoole\Coroutine\Channel(1)`        | Serialising access to an in-memory resource per worker      |
| `Semaphore`   | Single worker, coroutine-scoped       | `Swoole\Coroutine\Channel($permits)` | Capping concurrent access (e.g. outbound request pool)      |
| `File`        | Single host, cross-process            | `flock()`                            | Cron guards, shared-filesystem coordination                 |
| `Distributed` | Cross-host, cluster-wide              | Redis `SET NX EX` + Lua release      | Coordinating workers across machines                        |

## The interface

```php
namespace Utopia\Lock;

interface Lock
{
    public function acquire(float $timeout = 0.0): bool;
    public function tryAcquire(): bool;
    public function release(): void;

    /** @template T; @param callable(): T $callback; @return T */
    public function withLock(callable $callback, float $timeout = 0.0): mixed;
}
```

`withLock()` throws `Utopia\Lock\Exception\Contention` if the lock cannot be acquired before the timeout expires.

## Mutex

```php
use Swoole\Coroutine;
use Utopia\Lock\Mutex;
use function Swoole\Coroutine\run;

$mutex = new Mutex();

run(function () use ($mutex): void {
    for ($i = 0; $i < 8; $i++) {
        Coroutine::create(function () use ($mutex, $i): void {
            $mutex->withLock(function () use ($i): void {
                echo "worker {$i} holds the mutex\n";
                Coroutine::usleep(10_000);
            }, timeout: 5.0);
        });
    }
});
```

## Semaphore

```php
use Utopia\Lock\Semaphore;

$semaphore = new Semaphore(permits: 3);

$semaphore->withLock(function () {
    // at most three coroutines can be here at once
});
```

## File

```php
use Utopia\Lock\File;

$lock = new File('/var/run/my-daily-job.lock');

if (! $lock->tryAcquire()) {
    exit("another copy is already running\n");
}

try {
    runDailyJob();
} finally {
    $lock->release();
}
```

Pass `LOCK_SH` for shared (reader) mode:

```php
$readers = new File('/tmp/cache.lock', LOCK_SH);
$readers->withLock(fn () => readCache(), timeout: 1.0);
```

## Distributed

```php
use Redis;
use Utopia\Lock\Distributed;

$redis = new Redis();
$redis->connect('redis.internal', 6379);

$lock = new Distributed($redis, key: 'jobs:rebuild-index', ttl: 120);

$lock->setLogger(fn (string $message) => \error_log($message));

$lock->withLock(function () {
    rebuildSearchIndex();
}, timeout: 30.0);
```

Release is atomic: a Lua script verifies the lock value still matches this instance's token before deleting, so a lock that expires and is re-acquired elsewhere is never released by accident.

## Exception handling

```php
use Utopia\Lock\Exception;
use Utopia\Lock\Exception\Contention as ContentionException;

try {
    $lock->withLock($work, timeout: 5.0);
} catch (ContentionException) {
    // timed out trying to acquire
} catch (Exception) {
    // base class, catches anything thrown by this package
}
```

## Development

```bash
docker compose up -d
docker compose exec tests composer install
docker compose exec tests composer format:check
docker compose exec tests composer analyze
docker compose exec tests vendor/bin/phpunit
```

## License

MIT
