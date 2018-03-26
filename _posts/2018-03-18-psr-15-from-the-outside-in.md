---
layout: post
image: psr-15.png
---

I'm very glad the [PSR-15](https://www.php-fig.org/psr/psr-15/) specification has been approved. Standardizing middleware is very promising for the PHP ecosystem. In this article I will recapitulate what PSR-15 is about and present [a package I released](https://github.com/ellipsephp/handlers) to play with it.

![Psr-15](/images/psr-15.png)

## The PSR-15 specification

### Request handlers

First, the specification defines **request handlers**. Here is the interface:

```php
<?php

namespace Psr\Http\Server;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

interface RequestHandlerInterface
{
    public function handle(ServerRequestInterface $request): ResponseInterface;
}
```

So a request handler is basically an object producing a PSR-7 response from a given PSR-7 request. It can be seen as an abstraction of a whole web application:

```php
<?php

namespace App;

$app = new AppRequestHandler;

$response = $app->handle($request);
```

Obviously a single request handler is not very useful. The point is layers can progressively be added around a request handler in order to build a more complex application in a modular way. Those layers are defined as **middleware** by the specification.

### Middleware

Here is the interface for PSR-15 **middleware**:

```php
<?php

namespace Psr\Http\Server;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

interface MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface;
}
```

So a middleware also receive a PSR-7 request and return a PSR-7 response. The difference is the `->process()` method takes a request handler representing the web application as second parameter. Whether deeper middleware are wrapped around it or not is irrelevant from the middleware point of view. This way middleware can be used to act on the request the web application receive and on the response it produces, following one out of four patterns:

- Create a new PSR-7 request (PSR-7 requests are immutable) and delegate the response creation to the request handler
- Delegate the response creation to the request handler and return a new PSR-7 response (PSR-7 responses are immutable)
- A combination of the two points above
- Bypass the web application by creating a response on its own when a condition is met

Middleware can also let the response and request untouched but update a mutable service received through dependency injection.

Many things can be achieved with middleware. On the top of my head:

- Parsing the request body according to its content type
- Encrypt cookies
- Manage session data
- Catch exceptions and return an appropriate response
- Prevent access to resources
- Add general purpose variables to the template engine
- Choose among a set of request handlers and use it to produce a response (routing)

I think this is a beautiful pattern because it abstracts the essence of the web in a simple but modular way. Instead of relying on configuration values and conditionals, one can toggle functionalities by adding or removing middleware.

Yet tools are needed to manage the creation of those layered structures. I released such a tool with the [ellipse/handlers](https://githib.com/ellipsephp/handlers) package providing a minimal set of classes needed to easily dispatch a request through a list of PSR-15 middleware. Lets review it step by step in the next sections.

## The ellipse/handlers package

### Fallback request handler

First thing we need is to create the innermost request handler. It would be the last object producing a response when no other layer does. The `Ellipse\Handlers\FallbackRequestHandler` can be used for this purpose:

```php
<?php

namespace App;

use Ellipse\Handlers\FallbackRequestHandler;

// Any Psr-7 implementation can be used (using zend diactoros here)
use Zend\Diactoros\ServerRequestFactory;
use Zend\Diactoros\Response;

// Create an incoming request and a fallback response:
$request = ServerRequestFactory::fromGlobals();
$response = (new Response)->withStatus(404);

// The fallback request handler returns it:
$app = new FallbackRequestHandler($response);

// We now have an app returning a 404 response on every request!
$response = $app->handle($request);
```

We now need a way to wrap middleware around this useless application.

### Request handler decorator

The most basic way to wrap a middleware around a request handler is to use a decorator. The `Ellipse\Handlers\RequestHandlerWithMiddleware` class can be used to wrap a middleware around a request handler:

```php
<?php

// ...

use Ellipse\Handlers\RequestHandlerWithMiddleware;

// Use a router middleware to return a response before hitting the fallback:
$app = new FallbackRequestHandler($response);
$app = new RequestHandlerWithMiddleware($app, new RoutingMiddleware($router));

// The app now return a 404 response only when the router can't handle the request:
$response = $app->handle($request);
```

Of course `RequestHandlerWithMiddleware` implements `RequestHandlerInterface` and can also be decorated by another `RequestHandlerWithMiddleware`:

```php
<?php

// ...

use Ellipse\Handlers\RequestHandlerWithMiddleware;

// Remove the uri trailing slash before hitting the router:
$app = new FallbackRequestHandler($response);
$app = new RequestHandlerWithMiddleware($app, new RoutingMiddleware($router));
$app = new RequestHandlerWithMiddleware($app, new TrailingSlashMiddleware);

// Now the router catches either 'url/' or 'url':
$response = $app->handle($request);
```

As the decorator adds a middleware around a request handler, the one added the last is the first to process the request. Also it is a bit cumbersome to manually create many new `RequestHandlerWithMiddleware`.  A more intuitive way to manage lists of middleware is obviously needed.

### Middleware stack and queue

The `Ellipse\Handlers\RequestHandlerWithMiddlewareStack` class can be used to decorate a request handler with many middleware at once:

```php
<?php

// ...

use Ellipse\Handlers\RequestHandlerWithMiddlewareStack;

// Same as above. Still follow the LIFO order.
$app = new FallbackRequestHandler($response);

$app = new RequestHandlerWithMiddlewareStack($app, [
    new RoutingMiddleware($router),
    new TrailingSlashMiddleware,
]);

$response = $app->handle($request);
```

Here the middleware list is treated as a stack meaning they process the request in **LIFO** order (Last In First Out). Under the hood it just progressively decorates request handlers with `RequestHandlerWithMiddleware` , resulting in the same structure as in the previous example.

Another options is to use the `Ellipse\Handlers\RequestHandlerWithMiddlewareQueue` class:

```php
<?php

// ...

use Ellipse\Handlers\RequestHandlerWithMiddlewareQueue;

// Same as above but now follow the FIFO order.
$app = new FallbackRequestHandler($response);

$app = new RequestHandlerWithMiddlewareQueue($app, [
    new TrailingSlashMiddleware,
    new RoutingMiddleware($router),
]);

$response = $app->handle($request);
```

Here the middleware list is treated as a queue, meaning they process the request in **FIFO** order (First In First Out). This is usually the order developers are the more comfortable with, but you are free to use the approach you want.

## The ellipse/dispatcher package

The [ellipse/dispatcher](https://github.com/ellipsephp/dispatcher) package provides an `Ellipse\DispatcherFactory` class which can be used to produce `Ellipse\Dispatcher` instances. A `Dispatcher` works the same as a `RequestHandlerWithMiddlewareQueue`. It also have a `->with()` method producing a new `Dispatcher` with an additional middleware wrapped around the previous `Dispatcher`.

```php
<?php

// ...

use Ellipse\DispatcherFactory;

// Get a dispatcher factory.
$factory = new DispatcherFactory;

// Get a Dispatcher using the factory.
$dispatcher1 = $factory(new FallbackRequestHandler($response), [
    new TrailingSlashMiddleware,
    new RoutingMiddleware($router),
]);

// Works the same way as in the previous example.
$response = $dispatcher1->handle($request);

// Another middleware can be wrapped around the dispatcher.
$dispatcher2 = $dispatcher1->with(new ClientIpMiddleware);

// The ClientIpMiddleware is the first to process the request and receive
// $dispatcher1 as request handler.
$response = $dispatcher2->handle($request);
```

I'll show in future articles how `DispatcherFactory` can be used in conjonction with resolvers to use many different types of value as PSR-15 instances.

## Conclusion

Classes provided by the [ellipse/handlers](https://githib.com/ellipsephp/handlers) package can be used as a basic way to articulate PSR-15 instances together. More complex mechanisms are usually applied when dealing with lists of middleware, like resolving callables and container entries as actual middleware instances. I will present in future articles how this can be achieved with the [ellipse/dispatcher](https://githib.com/ellipsephp/dispatcher) package as a starting point.
