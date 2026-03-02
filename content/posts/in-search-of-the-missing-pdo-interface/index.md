---
date: '2026-03-01T00:00:00-05:00'
title: 'In Search of the Missing PDO Interface'
description: 'Why PDO has no interface after twenty years, how the ecosystem has worked around it, and what a minimal standard contract could look like.'
tags: ['PHP', 'Software Architecture']
---

PHP's PDO extension has been the standard database abstraction layer since PHP 5.1. It does its job well: it provides a unified API over multiple database drivers and handles the low-level details of connecting, querying, and fetching results. Yet, after nearly two decades, PDO is still missing something fundamental: an interface.

This might seem like a minor inconvenience, but the consequences ripple across the PHP ecosystem in ways that are worth examining. This post explores the gap, the workarounds the community has invented to fill it, and what a minimal standard interface could look like.

## The Problem with a Concrete Class

PDO is a native C extension implemented as a concrete class with no interface. This creates two practical problems.

The first is lazy loading. PDO connects to the database immediately in its constructor. If you want to defer the connection until the first actual query, a common need in applications that boot many services but only use some of them per request, you cannot do so without some form of indirection.

The second is substitution. Because PDO has no interface, any code that type-hints `PDO` directly cannot accept a decorator, a proxy, or a test double without resorting to inheritance. You cannot inject a logging wrapper, a retry decorator, or a mock without making it extend `PDO`, which forces you into a class hierarchy where composition would be far more appropriate.

## The Workarounds Are Not Pretty

The most common workaround is to extend PDO and override its constructor:

```php
class LazyPdo extends PDO
{
    private string $dsn;
    private string $user;
    private string $password;
    private array $options;

    public function __construct(string $dsn, string $user, string $password, array $options = [])
    {
        $this->dsn = $dsn;
        $this->user = $user;
        $this->password = $password;
        $this->options = $options;
        // do not call parent::__construct()
    }

    private function connect(): void
    {
        parent::__construct($this->dsn, $this->user, $this->password, $this->options);
    }
}
```

This works technically, but it is fragile. You must override every method that requires a connection to insert a call to `connect()` first. The class grows into a maintenance burden just to implement a concern that should be trivial.

More importantly, this approach still forces every consumer to type-hint `PDO`. You cannot swap in a test double that does not touch a database, or a decorator that logs queries, without applying the same inheritance trick again.

## The Ecosystem Has Noticed

The PHP community has not ignored this gap. It has worked around it, repeatedly and independently.

**Aura.Sql** (over 2.4 million downloads, actively maintained) ships its own `PdoInterface` extracted from its `ExtendedPdoInterface`. Its `AbstractExtendedPdo` extends `PDO` and overrides methods one by one to add lazy connection, query profiling, array quoting, and more. The result is a well-engineered but heavyweight class that grows new responsibilities over time precisely because the architecture forces everything into a single inheritance chain. Crucially, its `PdoInterface` lives in the `Aura\Sql` namespace -- no other library can depend on it without coupling themselves to Aura.Sql.

**lazypdo** (over 100,000 downloads) solves the lazy connection problem in isolation. It has not been updated since 2020. Users who depended on it now carry an abandoned package that will never be updated if PDO evolves.

**Doctrine DBAL** and **Nette Database** (millions of downloads between them) take a different approach entirely: they replace PDO with their own database abstraction. This is a legitimate architectural choice, but it is a heavy dependency to take on simply because PDO cannot be extended cleanly through composition.

The pattern here is familiar. Each library has independently identified the same gap and filled it with its own solution. Those solutions are incompatible with each other. A library that wants to accept any PDO-like object must either pick one of these abstractions, invent its own, or fall back to type-hinting `PDO` directly and accept the limitations that come with it.

This is precisely the situation that existed with HTTP messages before PSR-7. Symfony had `HttpFoundation`, Zend had its own request and response objects, and every other framework had theirs. PSR-7 did not replace those implementations. It provided the common contract around which the ecosystem could converge. Libraries that adopted `RequestInterface` and `ResponseInterface` became interoperable without depending on any specific framework.

All of these solutions point to the same need. Here is what a minimal standard interface could look like.

## A Minimal Interface

The proposed interface is deliberately minimal. It is a faithful mirror of PDO's public API, with one addition: a `getInstance()` method that returns the underlying `PDO` instance for the rare cases where a third-party library requires a concrete `PDO` object.

```php
interface PdoInterface
{
    public static function connect(string $dsn, ?string $username = null, ?string $password = null, ?array $options = null): PdoInterface;
    public static function getAvailableDrivers(): array;
    public function getInstance(): PDO;
    public function beginTransaction(): bool;
    public function commit(): bool;
    public function errorCode(): ?string;
    public function errorInfo(): array;
    public function exec(string $statement): int|false;
    public function getAttribute(int $attribute): mixed;
    public function inTransaction(): bool;
    public function lastInsertId(?string $name = null): string|false;
    public function prepare(string $query, array $options = []): PDOStatement|false;
    public function query(string $query, ?int $fetchMode = null): PDOStatement|false;
    public function quote(string $string, int $type = PDO::PARAM_STR): string|false;
    public function rollBack(): bool;
    public function setAttribute(int $attribute, mixed $value): bool;
}
```

This interface introduces nothing new. It does not add query builders, fetch helpers, or profiling hooks. It only captures what PDO already does. Any existing code that uses PDO already satisfies this contract, but cannot declare it.

## A Proxy as a Reference Implementation

With this interface, a lazy-loading proxy becomes straightforward. The implementation below accepts a callable that returns a PDO instance and defers the call until the first method is actually invoked:

```php
use PDO;
use RuntimeException;

class PdoProxy implements PdoInterface
{
    /** @var callable */
    private $resolver;

    private bool $resolved = false;

    private ?PDO $pdo = null;

    public function __construct(callable $resolver)
    {
        $this->resolver = $resolver;
    }

    public static function connect(string $dsn, ?string $username = null, ?string $password = null, ?array $options = null): PdoInterface
    {
        return new self(fn () => PDO::connect($dsn, $username, $password, $options));
    }

    public function getInstance(): PDO
    {
        if ($this->resolved) {
            return $this->pdo;
        }

        $object = ($this->resolver)();

        if (!$object instanceof PDO) {
            throw new RuntimeException('The callable did not return a PDO instance.');
        }

        $this->pdo = $object;
        $this->resolved = true;

        return $this->pdo;
    }

    public function beginTransaction(): bool
    {
        return $this->getInstance()->beginTransaction();
    }

    public function exec(string $statement): int|false
    {
        return $this->getInstance()->exec($statement);
    }

    public function prepare(string $query, array $options = []): PDOStatement|false
    {
        return $this->getInstance()->prepare($query, $options);
    }

    public function query(string $query, ?int $fetchMode = null): PDOStatement|false
    {
        return $this->getInstance()->query($query, $fetchMode);
    }

    // ... remaining methods follow the same pattern
}
```

The callable passed to the constructor can be anything: a closure that reads credentials from environment variables, a factory method on a container, or a simple instantiation. The proxy does not care. It only calls the resolver once, on first use, and caches the result.

Usage is transparent to the consumer:

```php
$pdo = new PdoProxy(function () use ($dsn, $username, $password, $options) {
    return new PDO($dsn, $username, $password, $options);
});

// No connection has been made yet.
// The connection is established here, on first use:
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
```

## What the Interface Unlocks

Having a shared interface opens the door to patterns that inheritance simply cannot support cleanly. Implementations of `PdoInterface` can introduce their own extensibility mechanisms (lazy loading, query profiling, retry logic) without forcing those concerns into a growing abstract class.

One natural fit is the middleware pattern, which will be familiar to anyone who has worked with PSR-15. Because `PdoInterface` provides a stable, typed contract, an implementation can expose a middleware stack where each layer receives `$next` typed as `PdoInterface`. For a detailed walkthrough of how to apply this pattern to any PHP library component, see [Using the Middleware Pattern to Extend PHP Libraries](https://maximegosselin.com/posts/using-the-middleware-pattern-to-extend-php-libraries/).

The point is not to prescribe how implementations should handle extensibility. It is that a standard interface makes those choices possible in the first place.

## A Non-Breaking Path Forward

A concern often raised against standardizing a PDO interface is the cost of migration. Libraries that currently type-hint `PDO` would need to change.

The transition does not have to be abrupt. PHP's union types offer a clean migration path:

```php
// Before
public function __construct(PDO $pdo) {}

// During transition
public function __construct(PDO|PdoInterface $pdo) {}

// After ecosystem adoption
public function __construct(PdoInterface $pdo) {}
```

Existing code that passes a `PDO` instance continues to work unchanged throughout the transition. No breaking changes are required at any stage.

If `PdoInterface` were eventually adopted at the language level, `PDO` itself would implement it natively. At that point, the union type `PDO|PdoInterface` would become redundant and libraries could drop directly to `PdoInterface` with no further changes.

## A PSR or a Language Feature?

There are two paths to standardizing this interface.

The first is a PSR. The [PHP-FIG workflow](https://www.php-fig.org/bylaws/psr-workflow/) requires a problem statement, a working group, and two independent trial implementations before acceptance. The problem statement is clear. The implementations already exist in the ecosystem: Aura.Sql and the proxy pattern described here would qualify as independent implementations. A PSR for `PdoInterface` would follow the same model as PSR-3 for loggers: a thin, stable contract that allows the ecosystem to converge without mandating any specific implementation. It would also solve a problem that community packages like lazypdo cannot: when PHP adds a new method to PDO, a PHP-FIG maintained interface would be updated in tandem, keeping all implementations in sync.

The second is a language-level solution. A future version of PHP could introduce `PdoInterface` as a native interface implemented by `PDO` itself. This would be the most impactful outcome: no external package, full native compatibility, and the ability for `PDO` to declare that it implements its own interface. PHP has done this before: `DateTimeInterface` was introduced in PHP 5.5 as a common contract for `DateTime` and `DateTimeImmutable`, allowing libraries to accept either without a concrete type-hint. There is no reason the same approach could not apply to PDO. This requires an RFC and a vote from PHP core contributors, which is a higher bar, but not an unreasonable one for a change this straightforward.

The two paths are not mutually exclusive. A PSR could serve as ecosystem validation before an RFC is proposed. If a standard `PdoInterface` package is widely adopted, the case for including it in the language becomes much stronger.

## An Invitation

This post is not a finished proposal. It is an exploration of a gap that has existed for a long time, filled imperfectly by a fragmented set of independent solutions.

The interface described here is not original, as variations of it exist in Aura.Sql, in lazypdo, and in other projects. What does not exist is a shared standard that libraries can depend on without coupling themselves to a specific implementation.

If you maintain a PHP library that accepts or returns PDO instances, does the absence of a standard interface affect your design decisions? If a PSR or RFC were proposed, would your project adopt it?

I would love to hear your thoughts. If there is enough interest in the community, the next step would be to bring this discussion to the [PHP-FIG](https://www.php-fig.org/get-involved/) and explore a formal proposal.