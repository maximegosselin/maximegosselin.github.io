---
title: "Proposing a PSR for PDO Providers"
date: 2026-03-10
description: "PdoInterface was the wrong solution. The real problem is that there is no standard contract for passing a PDO connection between libraries."
tags: ['PHP']
---

Last week I published [In Search of the Missing PDO Interface](/posts/in-search-of-the-missing-pdo-interface/), arguing that PDO should have an interface. The [discussion on r/PHP](https://www.reddit.com/r/PHP/comments/1rntvym/pdo_has_no_interface_after_20_years_does_it/) convinced me I was solving the wrong problem.

## The real problem

When you pass a `\PDO` instance to a library, you hand over a fixed connection, not access to one. The library now holds something it cannot reconnect if the connection drops, cannot swap out, and cannot wrap transparently. You have surrendered control over the connection lifecycle.

On top of that, every major database abstraction library (`doctrine/dbal`, `illuminate/database`, `nette/database`, `cakephp/database`) wraps PDO in its own connection object. If you need to pass a connection to another library, you must extract a `\PDO` snapshot from it. At that point, the DBAL no longer manages it: no reconnection, no lifecycle control.

A `PdoInterface` mirroring `\PDO` would address neither of these. It would be a large interface, it would still represent a fixed connection, and as [PHP internals attitudes](https://phpopendocs.com/internals/rfc_attitudes) make clear, anything solvable in userland has a low chance of being accepted as an RFC anyway.

## The solution: PdoProviderInterface

The right abstraction is not the connection itself, but the source of the connection.

PSR-20 introduced `ClockInterface` with a single method: `now()`. It does not abstract `\DateTimeImmutable`; it abstracts the source of the current time. The same pattern applies here. `PdoProviderInterface` exposes a single method:

```php
namespace Psr\Pdo;

interface PdoProviderInterface
{
    public function getConnection(): \PDO;
}
```

Consumers call `getConnection()` each time they need a connection. Implementers decide what that means: a lazy connection, a reconnecting one, a pooled one. The consumer knows nothing about that.

This also enables transparent interoperability. If Doctrine DBAL implements `PdoProviderInterface`, you can pass a DBAL connection directly to any library that accepts a `PdoProviderInterface`. No glue code, no snapshot leaking out of DBAL's control.

## The pre-draft PSR

I have written both documents:

- [Specification](https://raw.githubusercontent.com/maximegosselin/fig-standards/refs/heads/master/proposed/pdo-provider.md)
- [Meta document](https://raw.githubusercontent.com/maximegosselin/fig-standards/refs/heads/master/proposed/pdo-provider-meta.md)

The meta document covers the rationale, rejected alternatives, and example implementations. Start there.

A pull request against the PHP-FIG standards repository is open at https://github.com/php-fig/fig-standards/pull/1348.

## Feedback

If you have substantive feedback on the spec, leave it on the PR. For everything else, use the thread where this post was shared.
