# Middleware

> ℹ️ **New to middleware?**
>
> Middleware allows modifying the incoming request and outgoing response messages
and extracting this logic into reusable components.
This is frequently used for common functionality such as HTTP login, session handling, logging, and much more.

## Inline middleware functions

Middleware is any piece of logic that will wrap around your request handler.
You can add any number of middleware handlers to each route.
To get started, let's take a look at a basic middleware handler
by adding an additional callable before the final controller like this:

```php
<?php

$app->get(
    '/user',
    function (Psr\Http\Message\ServerRequestInterface $request, callable $next) {
        // optionally return response without passing to next handler
        // return new React\Http\Message\Response(403, [], "Forbidden!\n");
        
        // optionally modify request before passing to next handler
        // $request = $request->withAttribute('admin', false);
        
        // call next handler in chain
        $response = $next($request);
        assert($response instanceof Psr\Http\Message\ResponseInterface);
        
        // optionally modify response before returning to previous handler
        // $response = $response->withHeader('Content-Type', 'text/plain');
        
        return $response;
    },
    function (Psr\Http\Message\ServerRequestInterface $request) {
        $role = $request->getAttribute('admin') ? 'admin' : 'user';
        return new React\Http\Message\Response(200, [], "Hello $role!\n");
    }
);
```

This example shows how you could build your own middleware that can
modifying the incoming request and outgoing response messages alike.
Each middleware is responsible for calling the next handler in the chain or directly returning an error response if the request should not be processed.

## Middleware classes

While inline functions are easy to get started, it's easy to see how this would become a mess once you
keep adding more controllers to a single application.
For this reason, we recommend using middleware classes for production use-cases
like this:

```php
# src/DemoMiddleware.php
<?php

namespace Acme\Todo;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class DemoMiddleware
{
    public function __invoke(ServerRequestInterface $request, callable $next)
    {
        // optionally return response without passing to next handler
        // return new React\Http\Message\Response(403, [], "Forbidden!\n");

        // optionally modify request before passing to next handler
        // $request = $request->withAttribute('admin', false);

        // call next handler in chain
        $response = $next($request);
        assert($response instanceof ResponseInterface);

        // optionally modify response before returning to previous handler
        // $response = $response->withHeader('Content-Type', 'text/plain');

        return $response;
    }
}
```
```php
# app.php
<?php

use Acme\Todo\DemoMiddleware;
use Acme\Todo\UserController;

// …

$app->get('/user', new DemoMiddleware(), new UserController());
```

This highlights how middleware classes provide the exact same functionaly as using inline functions,
yet provide a cleaner and more reusable structure.
Accordingly, all examples below use middleware classes as the recommended style.

> ℹ️ **New to Composer autoloading?**
>
> This example uses namespaced classes as the recommended way
> in the PHP ecosystem. If you're new to setting up your project
> structure, see also [controller classes](../best-practices/controllers.md) for more details.

## Request middleware

To get started, we can add an example middleware handler that can modify the incoming request:

```php
# src/AdminMiddleware.php
<?php

namespace Acme\Todo;

use Psr\Http\Message\ServerRequestInterface;

class AdminMiddleware
{
    public function __invoke(ServerRequestInterface $request, callable $next)
    {
        $ip = $request->getServerParams()['REMOTE_ADDR'];
        if ($ip === '127.0.0.1') {
            $request = $request->withAttribute('admin', true);
        }

        return $next($request);
    }
}
```

```php
# src/UserController.php
<?php

namespace Acme\Todo;

use Psr\Http\Message\ServerRequestInterface;
use React\Http\Message\Response;

class UserController
{
    public function __invoke(ServerRequestInterface $request)
    {
        $role = $request->getAttribute('admin') ? 'admin' : 'user';
        return new Response(200, [], "Hello $role!\n");
    }
}
```

```php
# app.php
<?php

use Acme\Todo\AdminMiddleware;
use Acme\Todo\UserController;

// …

$app->get('/user', new AdminMiddleware(), new UserController());
```

For example, an HTTP `GET` request for `/user` would first call the middleware handler which then modifies this request and passes the modified request to the next controller function.
This is commonly used for HTTP authentication, login handling and session handling.

## Response middleware

Likewise, we can add an example middleware handler that can modify the outgoing response:

```php
# src/ContentTypeMiddleware.php
<?php

namespace Acme\Todo;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class ContentTypeMiddleware
{
    public function __invoke(ServerRequestInterface $request, callable $next)
    {
        $response = $next($request);
        assert($response instanceof ResponseInterface);
        
        return $response->withHeader('Content-Type', 'text/plain');
    }
}
```

```php
# src/UserController.php
<?php

namespace Acme\Todo;

use Psr\Http\Message\ServerRequestInterface;
use React\Http\Message\Response;

class UserController
{
    public function __invoke(ServerRequestInterface $request)
    {
        $name = 'Alice';
        return new Response(200, [], "Hello $name!\n");
    }
}
```

```php
# app.php
<?php

use Acme\Todo\ContentTypeMiddleware;
use Acme\Todo\UserController;

// …

$app->get('/user', new ContentTypeMiddleware(), new UserController());
```

For example, an HTTP `GET` request for `/user` would first call the middleware handler which passes on the request to the controller function and then modifies the response that is returned by the controller function.
This is commonly used for cache handling and response body transformations (compression etc.).

## Async middleware

One of the core features of X is its async support.
As a consequence, each middleware handler can also return
[promises](../async/promises.md) or [coroutines](../async/coroutines.md).
While [request middleware](#request-middleware) doesn't usually have to care
about async responses, this particularly affects
[response middleware](#response-middleware) that wants to change the outgoing
response.

Here's an example middleware handler that can modify the outgoing response no
matter whether the next request handler returns a
[promise](../async/promises.md), a [coroutine](../async/coroutines.md) or
a response object synchronously:

```php
# src/AsyncAwareContentTypeMiddleware.php
<?php

namespace Acme\Todo;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use React\Promise\PromiseInterface;

class AsyncContentTypeMiddleware
{
    public function __invoke(ServerRequestInterface $request, callable $next)
    {
        $response = $next($request);

        if ($response instance PromiseInterface) {
            return $response->then(function (ResponseInterface $response) {
                return $this->handle($response);
            });
        } elseif ($response instanceof \Generator) {
            return (function () use ($response) {
                return $this->handle(yield from $response);
            })();
        } else {
            return $this->handle($response);
        }
    }

    private function handle(ResponseInterface $response): ResponseInterface
    {
        return $response->withHeader('Content-Type', 'text/plain');
    }
}
```
```php
# src/AsyncUserController.php
<?php

namespace Acme\Todo;

use Psr\Http\Message\ServerRequestInterface;
use React\EventLoop\Loop;
use React\Http\Message\Response;
use React\Promise\Promise;
use React\Promise\PromiseInterface;

class AsyncUserController
{
    public function __invoke(ServerRequestInterface $request): \Generator
    {
        // async pseudo code to load some data from an external source
        $promise = $this->fetchRandomUserName();

        $name = yield $promise;
        assert(is_string($name));

        return new Response(200, [], "Hello $name!\n");
    }

    /**
     * @return PromiseInterface<string>
     */
    private function fetchRandomUserName(): PromiseInterface
    {
        return new Promise(function ($resolve) {
            Loop::addTimer(0.01, function () use ($resolve) {
                $resolve('Alice');
            });
        });
    }
}
```
```php
# app.php
<?php

use Acme\Todo\AsyncContentTypeMiddleware;
use Acme\Todo\AsyncUserController;

// …

$app->get('/user', new AsyncContentTypeMiddleware(), new AsyncUserController());
```

For example, an HTTP `GET` request for `/user` would first call the middleware handler which passes on the request to the controller function and then modifies the response that is returned by the controller function.
This is commonly used for cache handling and response body transformations (compression etc.).

> 🔮 **Future fiber support in PHP 8.1**
>
> In the future, PHP 8.1 will provide native support for [fibers](../async/fibers.md).
> Once fibers become mainstream, there would be little reason to use
> Generator-based coroutines anymore.
> While fibers will help to avoid using promises for many common use cases,
> promises will still be useful for concurrent execution.
> See [fibers](../async/fibers.md) for more details.

## Global middleware

Additionally, you can also add middleware to the [`App`](app.md) object itself
to register a global middleware handler:

```php hl_lines="7"
# app.php
<?php

use Acme\Todo\AdminMiddleware;
use Acme\Todo\UserController;

$app = new FrameworkX\App(new AdminMiddleware());

$app->get('/user', new UserController());

$app->run();
```

Any global middleware handler will always be called for all registered routes
and also any requests that can not be routed.

You can also combine global middleware handlers (think logging) with additional
middleware handlers for individual routes (think authentication).
Global middleware handlers will always be called before route middleware handlers.
