---
date: '2026-05-04T00:00:00-05:00'
title: 'Bootstrapping a Frameworkless PHP Application'
description: 'A single bootstrap file. A PSR-11 container. No fullstack framework required.'
tags: ['PHP', 'Architecture']
---

# Bootstrapping a Frameworkless PHP Application

> The source code for this article is available at [github.com/maximegosselin/php-bootstrap](https://github.com/maximegosselin/php-bootstrap)

Laravel and Symfony are well-engineered, but they make decisions for your entire application: routing, authentication, ORM, templating, event dispatching, and so on. The PHP ecosystem is now rich enough that this tradeoff is optional. Quality libraries exist for every layer, each doing one thing well, assembled into exactly what you need. That still leaves the glue work: bootstrapping, autoloading, configuration, wiring services together. You can do it yourself with a single file.

## Anatomy of `bootstrap.php`

[`bootstrap.php`](https://github.com/maximegosselin/php-bootstrap/blob/main/bootstrap.php) is the backbone of the application. Every entry point (web, CLI, script, tests) requires it, gets back a PSR-11 container with access to the full service graph, and pulls what it needs from there. The entire bootstrap fits in a single file at the root of your project:

```php
<?php

declare(strict_types=1);

use App\ServiceProvider;
use Dotenv\Dotenv;
use Dotenv\Repository\Adapter\EnvConstAdapter;
use Dotenv\Repository\Adapter\PutenvAdapter;
use Dotenv\Repository\RepositoryBuilder;
use League\Container\Container;
use League\Container\ReflectionContainer;
use Psr\Container\ContainerInterface;

return (function (): ContainerInterface {

    // 1. Set the working directory
    chdir(__DIR__);

    // 2. Initialize the autoloader
    require 'vendor/autoload.php';

    // 3. Load configuration
    $repository = RepositoryBuilder::createWithNoAdapters()
        ->addAdapter(EnvConstAdapter::class)
        ->addWriter(PutenvAdapter::class)
        ->immutable()
        ->make();
    Dotenv::create($repository, __DIR__)->safeLoad();
    $config = include('config.php');

    // 4. Build and return the container
    return new Container()
        ->delegate(new ReflectionContainer(true))
        ->addServiceProvider(new ServiceProvider($config));
})();
```

These four steps are the minimum. The bootstrap could also configure PHP with `ini_set`, set up `gettext` for localization, register a global error handler, or handle any other initialization that needs to happen once, before anything else runs.

### 1. Set the working directory

```php
chdir(__DIR__);
```

No matter how the application is invoked (a web request, a cron job, a test suite), the working directory is anchored to the project root. Relative paths work everywhere.

### 2. Initialize the autoloader

```php
require 'vendor/autoload.php';
```

Standard Composer PSR-4 autoloading.

### 3. Load configuration

Environment variables are loaded from `.env` using [vlucas/phpdotenv](https://github.com/vlucas/phpdotenv). [`config.php`](https://github.com/maximegosselin/php-bootstrap/blob/main/config.php) then translates them into a typed PHP array. In this application, those values control which calculator operations are available:

```php
$bool = fn(string $var): bool => filter_var(getenv($var) ?? true, FILTER_VALIDATE_BOOL);

return [
    'allow_add'      => $bool('ALLOW_ADD'),
    'allow_subtract' => $bool('ALLOW_SUBTRACT'),
    'allow_multiply' => $bool('ALLOW_MULTIPLY'),
    'allow_divide'   => $bool('ALLOW_DIVIDE'),
];
```

That array flows into [`ServiceProvider`](https://github.com/maximegosselin/php-bootstrap/blob/main/src/ServiceProvider.php), which injects the relevant values into each service. Behaviour is controlled through `.env` without touching the code.

### 4. Build and return the container

The entire file is wrapped in an IIFE (Immediately Invoked Function Expression) that returns a configured [league/container](https://container.thephpleague.com/) instance: a `ReflectionContainer` delegate for autowiring, and a `ServiceProvider` that registers explicit bindings. The caller captures it with a plain assignment:

```php
$container = require 'path/to/bootstrap.php';
```

Intermediate variables like `$config` stay scoped inside the anonymous function and disappear when it returns. Nothing leaks into the global namespace.

You could use any other PSR-11 compatible container. The entry points would not change.

## Four Ways to Invoke the Application

To make this concrete, I built a simple PHP calculator application that supports the four basic operations: addition, subtraction, multiplication, and division. It is intentionally minimal. The point is the bootstrap, not the business logic.

Every entry point loads `bootstrap.php` and captures the returned container.

### 1. Web

See [`/public/index.php`](https://github.com/maximegosselin/php-bootstrap/blob/main/public/index.php)

```php
<?php

declare(strict_types=1);

use App\Web\CalcRequestHandler;
use Psr\Container\ContainerInterface;
use Slim\Factory\AppFactory;
use Slim\Handlers\Strategies\RequestHandler;

/** @var ContainerInterface $container */
$container = require __DIR__ . '/../bootstrap.php';

AppFactory::setContainer($container);
$app = AppFactory::create();
$app->getRouteCollector()->setDefaultInvocationStrategy(new RequestHandler(true));
$app->addRoutingMiddleware();
$app->addErrorMiddleware(true, false, false);
$app->addBodyParsingMiddleware();

$app->get('/calc', CalcRequestHandler::class);

$app->run();
```

Slim handles routing and middleware. But Slim is a detail. The container is framework-agnostic. Any HTTP library or router would plug in the same way.

**How to invoke**

```bash
cd public && php -S localhost:3000
```

[http://localhost:3000/calc?n1=5&op=add&n2=3](http://localhost:3000/calc?n1=5&op=add&n2=3)

### 2. CLI

See [`/console`](https://github.com/maximegosselin/php-bootstrap/blob/main/console)

```php
#!/usr/bin/env php
<?php

declare(strict_types=1);

use App\Console\Console;
use Psr\Container\ContainerInterface;

(function () {
    /** @var ContainerInterface $container */
    $container = include __DIR__ . '/bootstrap.php';

    new Console($container)->run();
})();
```

Symfony Console handles argument parsing and command dispatch. It could be `getopt()`. The container does not care.

**How to invoke**

```bash
./console calc 5 add 3
```

### 3. Standalone script

See [`/bin/sums.php`](https://github.com/maximegosselin/php-bootstrap/blob/main/bin/sums.php)

```php
<?php

declare(strict_types=1);

use App\Service\CalculatorInterface;
use App\Service\Operation;
use Psr\Container\ContainerInterface;

/** @var ContainerInterface $container */
$container = require __DIR__ . '/../bootstrap.php';

/** @var CalculatorInterface $calculator */
$calculator = $container->get(CalculatorInterface::class);

foreach (range(1, 10) as $n1) {
    foreach (range(1, 10) as $n2) {
        echo sprintf('%d + %d = %d', $n1, $n2, $calculator->calculate($n1, Operation::ADD, $n2)) . PHP_EOL;
    }
}
```

One `require`, and the full service graph is available. No framework, no boilerplate. This is the pattern for one-off scripts, data migrations, maintenance jobs: anything that needs access to your services without going through HTTP or a CLI framework.

**How to invoke**

```bash
php bin/sums.php
```

### 4. Tests

See [`/tests/BaseTestCase.php`](https://github.com/maximegosselin/php-bootstrap/blob/main/tests/BaseTestCase.php)

```php
<?php

namespace Test;

use PHPUnit\Framework\TestCase;
use Psr\Container\ContainerInterface;

abstract class BaseTestCase extends TestCase
{
    protected ContainerInterface $container;

    public function setUp(): void
    {
        parent::setUp();
        $this->container = include 'bootstrap.php';
    }
}
```

See [`/tests/CalculatorTest.php`](https://github.com/maximegosselin/php-bootstrap/blob/main/tests/CalculatorTest.php)

```php
<?php

namespace Test;

use App\Service\CalculatorInterface;
use App\Service\Operation;
use PHPUnit\Framework\Attributes\Test;

class CalculatorTest extends BaseTestCase
{
    private CalculatorInterface $calculator;

    public function setUp(): void
    {
        parent::setUp();
        $this->calculator = $this->container->get(CalculatorInterface::class);
    }

    #[Test]
    public function add(): void
    {
        $this->assertEquals(
            4 + 6,
            $this->calculator->calculate(4, Operation::ADD, 6),
        );
    }

    // ...
}
```

The test base class bootstraps the real container before each test. Tests run against the actual service graph. No mocks at the container level, no divergence between what you test and what you ship.

**How to invoke**

```bash
./vendor/bin/phpunit
```

Configuration can be controlled independently for tests via `phpunit.xml`, without touching the `.env` file:

```xml
<php>
    <env name="ALLOW_ADD" value="true"/>
    <env name="ALLOW_SUBTRACT" value="true"/>
    <env name="ALLOW_MULTIPLY" value="true"/>
    <env name="ALLOW_DIVIDE" value="true"/>
</php>
```

## Longevity and Responsibility

Every line in [`composer.json`](https://github.com/maximegosselin/php-bootstrap/blob/main/composer.json) for this application is intentional. No hidden dependencies on a framework's HTTP kernel, ORM, or event dispatcher. You know exactly what is there and why.

When a new PHP version ships, each library can be updated independently. If one falls behind or is abandoned, you replace just that piece. You are never blocked by a fullstack framework's release cycle.

The real argument for the frameworkless approach is not purism or minimalism. It is ownership: knowing exactly what your application depends on, why each dependency is there, and how to replace it if needed.
