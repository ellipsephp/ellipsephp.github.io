---
layout: post
image: resolving.png
---

In a previous article I explained how to use classes from [ellipse/handlers](https://github.com/ellipsephp/handlers) and [ellipse/dispatcher](https://github.com/ellipsephp/dispatcher) packages as a basic way to dispatch a request through a list of [PSR-15](https://www.php-fig.org/psr/psr-15/) instances. Here we will add flexibility by using callables and class names as PSR-15 middleware and request handler using other [ellipse packages](https://github.com/ellipsephp).

![Resolving](/images/resolving.png)

## Resolving callables

It can be useful to use callables as middleware and request handlers for small applications or for quick prototyping.

The packages [ellipse/middleware-callable](https://github.com/ellipsephp/middleware-callable) and [ellipse/handlers-callable](https://github.com/ellipsephp/handlers-callable) provide implementations of PSR-15 middleware and request handler using a callable to produce a response.

```php
<?php

namespace App;

use Ellipse\DispatcherFactory;
use Ellipse\Middleware\CallableMiddleware;
use Ellipse\Handlers\CallableRequestHandler;

// Using zend diactoros as Psr-7 implementation.
use Zend\Diactoros\ServerRequestFactory;
use Zend\Diactoros\Response\HtmlResponse;

// Get the incoming request.
$request = ServerRequestFactory::fromGlobals();

// Get a dispatcher factory.
$factory = new DispatcherFactory;

// This middleware uses the given callable to produce a response.
$middleware = new CallableMiddleware(function ($request, $handler) {

    if ($request->getUri()->getPath() === '/hello') {

        return new HtmlResponse('hello world');

    }

    return $handler->handle($request);

});

// This request handler uses the given callable to produce a response.
$fallback = new CallableRequestHandler(function ($request) {

    return new HtmlResponse('Not found', 404);

});

// Returns hello world response when url path is /hello, not found response otherwise.
$response = $factory($fallback, [$middleware])->handle($request);
```

This is useful yet it would be even easier to not have to instantiate `CallableMiddleware` and `CallableRequestHandler` for every callables.

For this purpose, the [ellipse/dispatcher-callable](https://github.com/ellipsephp/dispatcher-callable) provides an `Ellipse\Dispatcher\ResolverCallable` class which can decorate any implementation of `Ellipse\DispatcherFactoryInterface`. This decorated `DispatcherFactory` automatically wraps `CallableMiddleware` and `CallableRequestHandler` instances around callables used when creating a `Dispatcher`.

```php
<?php

// ...

use Ellipse\Dispatcher\CallableResolver;

// This factory produces instances of Dispatcher with CallableMiddleware and
// CallableRequestHandler instances wrapped around callables.
$factory = new CallableResolver(new DispatcherFactory);

// A CallableMiddleware instance will be wrapped around this callable.
$middleware = function ($request, $handler) {

    if ($request->getUri()->getPath() === '/hello') {

        return new HtmlResponse('hello world');

    }

    return $handler->handle($request);

};

// A CallableRequestHandler instance will be wrapped around this callable.
$fallback = function ($request) {

    return new HtmlResponse('Not found', 404);

};

// Works the same as in the previous example.
$response = $factory($fallback, [$middleware])->handle($request);
```

## Resolving class names

Middleware and request handler instances often need a complex building logic to inject all their dependencies. Also using a router can result in building different middleware stacks according to the incoming request so it is useless to instantiate all middleware on every requests. Those issues can be solved by registering middleware and request handlers in a PSR-11 container and to retrieve them only when they are actually needed.

The packages [ellipse/middleware-container](https://github.com/ellipsephp/middleware-container) and [ellipse/handlers-container](https://github.com/ellipsephp/handlers-container) provide implementations of PSR-15 middleware and request handler using a PSR-11 container and a container entry id. When processing/handling a request the container entry is retrieved and the response creation is delegated to its own `process`/`handle` method.

For example here is a middleware depending on an hypothetic `App\TemplateEngineInterface` implementation:

```php
<?php

namespace App\Middleware;

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

use Zend\Diactoros\Response\HtmlResponse;

use App\TemplateEngineInterface;

class HelloMiddleware implements MiddlewareInterface
{
    private $engine;

    public function __construct(TemplateEngineInterface $engine)
    {
        $this->engine = $engine;
    }

    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        if ($request->getUri()->getPath() === '/hello') {

            $contents = $this->engine->render('hello');

            return new HtmlResponse($contents);
        }

        return $handler->handle($request);
    }
}
```

This request handler too:

```php
<?php

namespace App\Handlers;

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Server\RequestHandlerInterface;

use Zend\Diactoros\Response\HtmlResponse;

use App\TemplateEngineInterface;

class NotFoundRequestHandler implements RequestHandlerInterface
{
    private $engine;

    public function __construct(TemplateEngineInterface $engine)
    {
        $this->engine = $engine;
    }

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $contents = $this->engine->render('notfound');

        return new HtmlResponse($contents, 404);
    }
}
```

Obviously it is useless to build instances of those two classes (including the template engine one) when they are not needed to handle the incoming request. Factories for those two classes can be registered in a PSR-11 container:

```php
<?php

namespace App;

use SomePsr11Container;
use App\Middleware\HelloMiddleware;
use App\Handlers\NotFoundRequestHandler;

$container = new SomePsr11Container;

$container->set(TemplateEngineInterface::class, function () {

    return new TemplateEngine('templates_path');

});

$container->set(HelloMiddleware::class, function ($container) {

    $engine = $container->get(TemplateEngineInterface::class);

    return new HelloMiddleware($engine);

});

$container->set(NotFoundRequestHandler::class, function ($container) {

    $engine = $container->get(TemplateEngineInterface::class);

    return new NotFoundRequestHandler($engine);

});
```

And finally those container entries can be used by instances of `Ellipse\Middleware\ContainerMiddleware` and `Ellipse\Handlers\ContainerRequestHandler`:

```php
<?php

namespace App;

use Ellipse\DispatcherFactory;
use Ellipse\Middleware\ContainerMiddleware;
use Ellipse\Handlers\ContainerRequestHandler;

use App\Middleware\HelloMiddleware;
use App\Handlers\NotFoundRequestHandler;

// Using zend diactoros as Psr-7 implementation.
use Zend\Diactoros\ServerRequestFactory;

// Get the incoming request.
$request = ServerRequestFactory::fromGlobals();

// Get a dispatcher factory.
$factory = new DispatcherFactory;

// This middleware uses the HelloMiddleware::class entry of the container
// to produce a response.
$middleware = new ContainerMiddleware($container, HelloMiddleware::class);

// This request handler uses the NotFoundRequestHandler::class entry of the container
// to produce a response.
$fallback = new ContainerRequestHandler($container, NotFoundRequestHandler::class);

// Returns hello world response when url path is /hello, not found response otherwise.
// NotFoundRequestHandler instance is _not_ retrieved from the container when the middleware
// returns a response.
$response = $response($fallback, [$middleware])->handle($request);
```

As with the callable resolving example, instances of `ContainerMiddleware` and `ContainerRequestHandler` can be automatically wrapped around middleware and request handler class names using the `Ellipse\Dispatcher\ContainerResolver` class from the [ellipse/dispatcher-container](https://github.com/ellipsephp/dispatcher-container) package.

```php
<?php

// ...

use Ellipse\Dispatcher\ContainerResolver;

// This factory produces instances of Dispatcher with:
// - ContainerMiddleware instances wrapped around middleware class names
// - ContainerRequestHandler instances wrapped around request handler class names
$factory = new ContainerResolver($container, new DispatcherFactory);

// Works the same as in the previous example.
$dispatcher = $response(NotFoundRequestHandler::class, [
    HelloMiddleware::class,
]);

$response = $dispatcher->handle($request);
```

## Resolving multiple types of value

Of course, as `CallableResolver` and `ContainerResolver` both implements and decorates `Ellipse\DispatcherFactoryInterface`, they can be used together to resolve both callables and class names:


```php
<?php

// ...

use Ellipse\Dispatcher\CallableResolver;
use Ellipse\Dispatcher\ContainerResolver;

// This factory produces instances of Dispatcher with:
// - CallableMiddleware and CallableRequestHandler instances wrapped around callables
// - ContainerMiddleware instances wrapped around middleware class names
// - ContainerRequestHandler instances wrapped around request handler class names
$factory = new ContainerResolver($container,
    new CallableResolver(
        new DispatcherFactory
    )
);

// Works the same as in the previous example.
$middleware = function ($request, $handler) {

    if ($request->getUri()->getPath() === '/hello') {

        return new HtmlResponse('hello world');

    }

    return $handler->handle($request);

};

$dispatcher = $response(NotFoundRequestHandler::class, [$middleware]);

$response = $dispatcher->handle($request);
```

## Conclusion

Dispatcher factory decorators can be used to add flexibility when building PSR-15 dispatchers. From here the possibility are endless, anyone can create a dispatcher factory decorator implementing `DispatcherFactoryInterface` to create `Dispatcher` instances from different types of value. I will present in future articles how to use [ellipse packages](https://github.com/ellipsephp) to build dispatchers using controller methods or ADR actions as request handlers.
