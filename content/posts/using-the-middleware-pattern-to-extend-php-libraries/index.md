---
date: '2026-02-27T00:00:00-05:00'
title: 'Using the Middleware Pattern to Extend PHP Libraries'
description: 'How to apply the PSR-15 middleware pattern to any PHP library component, as an alternative to the Decorator pattern.'
tags: ['PHP', 'Software Architecture']
---

If you have been writing PHP professionally for the past few years, you have almost certainly encountered PSR-15. It formalized the concept of middleware for HTTP request handling, and it changed the way we think about extensibility in PHP applications.

But here is the thing: the idea behind PSR-15 is too good to leave confined to HTTP.

This article is for PHP developers who maintain libraries. If you have ever exposed an extension point in your library using the Decorator pattern, I want to show you a better default.

---

## What PSR-15 Got Right

PSR-15 defines a `MiddlewareInterface` with a single method:

```php
public function process(
    ServerRequestInterface $request,
    RequestHandlerInterface $handler,
): ResponseInterface;
```

The brilliance is in the `$handler` parameter. It gives the middleware a typed reference to the next layer in the chain. The middleware can execute logic before calling `$handler->handle()`, after, both, or not at all (short-circuit). Multiple middlewares compose into an onion: each layer wraps the next, and execution flows symmetrically in and out.

This model is dynamic. You can add middlewares at runtime, conditionally, from configuration, from a service container. The stack is assembled and can be modified at any point.

---

## The Problem with Decorators

The Decorator pattern is a valid tool. It is not wrong. But as a library extension mechanism, it has a significant limitation: it is static.

A Decorator is configured once, at instantiation time, by wrapping one object inside another:

```php
$renderer = new CachingRenderer(
    new PageTitleRenderer(
        new CoreRenderer()
    )
);
```

Once built, the chain is fixed. You cannot add a layer at runtime. You cannot enable or disable a layer based on a condition that emerges later. And if your user wants to add their own layer, they have to wrap the whole thing again from the outside, with no guarantee of where in the chain their logic will execute.

Middleware solves all of this.

---

## Applying the Pattern to Any Component

The pattern translates directly to any component: the middleware interface mirrors the component interface exactly, with `$next` (typed as the component interface) added as the last parameter of each method. Everything stays typed, IDE-friendly, and explicit.

Let's build a `HtmlRenderer` component from scratch using this approach.

### The Component Interface

`HtmlRenderer` takes a template name and a data array, merges them, and returns the rendered HTML as a string.

```php
interface HtmlRendererInterface
{
    public function render(string $template, array $data = []): string;
}
```

### The Middleware Interface

The middleware interface mirrors the component interface exactly, with one addition: `$next` as the last parameter.

```php
interface MiddlewareInterface
{
    public function render(
        string $template,
        array $data = [],
        HtmlRendererInterface $next,
    ): string;
}
```

This is the key design decision. Because `$next` is typed as `HtmlRendererInterface`, your IDE knows exactly what methods are available on it.

### The Middleware Delegator

The `MiddlewareDelegator` is the glue that makes the chain work. It implements `HtmlRendererInterface` and bridges the gap between the component interface and the middleware interface:

```php
class MiddlewareDelegator implements HtmlRendererInterface
{
    private MiddlewareInterface $middleware;

    private HtmlRendererInterface $next;

    public function __construct(MiddlewareInterface $middleware, HtmlRendererInterface $next)
    {
        $this->middleware = $middleware;
        $this->next = $next;
    }

    public function render(string $template, array $data = []): string
    {
        return $this->middleware->render($template, $data, $this->next);
    }
}
```

### The HtmlRenderer

`HtmlRenderer` is the public facade. It holds the middleware stack and delegates to it. The `Core` class handles the actual rendering; its implementation details are irrelevant here.

```php
class HtmlRenderer implements HtmlRendererInterface
{
    private Core $core;

    /** @var MiddlewareInterface[] */
    private array $middlewares = [];

    private HtmlRendererInterface $chain;

    public function __construct(Core $core)
    {
        $this->core = $core;
        $this->chainMiddlewares();
    }

    public function render(string $template, array $data = []): string
    {
        return $this->chain->render($template, $data);
    }

    public function addMiddleware(MiddlewareInterface $middleware): void
    {
        $this->middlewares[] = $middleware;
        $this->chainMiddlewares();
    }

    private function chainMiddlewares(): void
    {
        $this->chain = array_reduce(
            $this->middlewares,
            fn (HtmlRendererInterface $carry, MiddlewareInterface $item): HtmlRendererInterface =>
                new MiddlewareDelegator($item, $carry),
            $this->core,
        );
    }
}
```

The chain is built with `array_reduce`. The `Core` is the starting value. Each middleware wraps the previous layer inside a `MiddlewareDelegator`. The last middleware added becomes the outermost layer.

---

## Two Concrete Middlewares

### PageTitleMiddleware: Modifying the Input

This middleware injects a `pageTitle` into `$data` based on the template's section prefix, before passing to the next layer.

```php
class PageTitleMiddleware implements MiddlewareInterface
{
    private const array SECTION_TITLES = [
        'admin'   => 'Admin',
        'account' => 'My Account',
        'shop'    => 'Shop',
        'blog'    => 'Blog',
    ];

    public function render(
        string $template,
        array $data = [],
        HtmlRendererInterface $next,
    ): string {
        $prefix = explode('-', $template)[0];

        if (isset(self::SECTION_TITLES[$prefix])) {
            $data['pageTitle'] = self::SECTION_TITLES[$prefix];
        }

        return $next->render($template, $data);
    }
}
```

The middleware calls `$next->render()` with the enriched `$data`. The rest of the chain, including `Core`, sees the `pageTitle` as if it had always been there.

### CacheMiddleware: Short-Circuiting the Chain

This middleware computes a hash from the template name and the (already enriched) data, then checks the cache before calling `$next`. If there is a cache hit, `$next` is never called at all.

```php
use Psr\Cache\CacheItemPoolInterface;

class CacheMiddleware implements MiddlewareInterface
{
    private CacheItemPoolInterface $cache;

    private int $ttl;

    public function __construct(CacheItemPoolInterface $cache, int $ttl = 3600)
    {
        $this->cache = $cache;
        $this->ttl = $ttl;
    }

    public function render(
        string $template,
        array $data = [],
        HtmlRendererInterface $next,
    ): string {
        $key = hash('sha256', $template . serialize($data));
        $item = $this->cache->getItem($key);

        // Cache hit: short-circuit the chain
        if ($item->isHit()) {
            return $item->get();
        }

        $result = $next->render($template, $data);

        // Cache miss: store the result for next time
        $item->set($result);
        $item->expiresAfter($this->ttl);
        $this->cache->save($item);

        return $result;
    }
}
```

The ability to short-circuit is something the Decorator pattern cannot express cleanly. Here it is a natural consequence of whether you call `$next` or not.

---

## Before, After, and Both

These two middlewares illustrate three fundamental execution models.

`PageTitleMiddleware` is a **Before** middleware: all its logic runs before delegating to `$next`. It enriches the input and steps aside.

```php
// PageTitleMiddleware::render()
// Before: logic runs before $next
$data['pageTitle'] = self::SECTION_TITLES[$prefix];
return $next->render($template, $data);
```

`CacheMiddleware` is a **Before + After** middleware: it checks the cache before calling `$next`, and stores the result after. The call to `$next` is wrapped on both sides.

```php
// CacheMiddleware::render()
// Before: check cache
if ($item->isHit()) {
    return $item->get();
}

// After: store result
$result = $next->render($template, $data);
$this->cache->save($item);
return $result;
```

This is the same model as PSR-15. The only difference is that `$next` is typed as `HtmlRendererInterface` instead of `RequestHandlerInterface`.

A middleware can also be **After** only, executing logic exclusively on the result returned by `$next`, without touching the input at all. An HTML minifier would be a good example: call `$next`, then process the output.

---

## Wiring It Together

```php
$renderer = new HtmlRenderer(new Core());

$renderer->addMiddleware(new CacheMiddleware($cache, 600));
$renderer->addMiddleware(new PageTitleMiddleware());
```

The following diagram illustrates how the middlewares wrap around the component, forming the onion layers:

![](middleware-stack.svg)

Execution order for a cache miss:

```
PageTitleMiddleware → CacheMiddleware → Core
```

`PageTitleMiddleware` enriches `$data` with the page title first. `CacheMiddleware` then computes its hash on the enriched data and caches the final HTML output. On the next identical request, the chain short-circuits at `CacheMiddleware` and `PageTitleMiddleware` is never called.

Middlewares can also be added at runtime, conditionally:

```php
if ($debugMode) {
    $renderer->addMiddleware(new ProfilingMiddleware($logger));
}
```

No rebuilding of the object graph. No wrapping from the outside. Just add the middleware and the chain rebuilds itself.

---

## What This Gives Your Users

When you design your library with middleware support from the start, your users get:

- **Input modification**: inject, transform or validate data before it reaches the core
- **Output modification**: post-process the result without touching the core
- **Short-circuiting**: skip the core entirely when appropriate (caching, mocking, feature flags)
- **Runtime composition**: add or configure behaviour based on conditions that emerge after instantiation
- **Typed extension points**: `$next` is the component interface, not a callable; autocomplete works, refactoring works

The implementation cost is low: one interface, one delegator class, one `array_reduce`. The extensibility you hand to your users is disproportionately large.

---

The pattern you just saw, one interface, one delegator, one `array_reduce`, is all it takes.

PSR-15 did not invent middleware. But it showed the PHP community what a well-designed, typed middleware interface looks like. There is no reason to leave that idea at the HTTP layer.

If you maintain a PHP library with any non-trivial processing, consider building middleware support in from day one. Your users will thank you, and so will your future self.
