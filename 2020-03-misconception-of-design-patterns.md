# The misconception of Design Patterns

<p align="center">
<img src="https://i.redd.it/f1vw64ete8i31.jpg" alt="Say 'anti-pattern' again, I dare you, I double dare you motherfucker!" width="640" height="360">
</p>
<p align="center">
Say 'anti-pattern' again, I dare you, I double dare you motherfucker!
</p> 

## What I have in mind when someone mentions anti-pattern
You want to put a nail into the wall. You choose a hammer to do the job.  
And then some one comes around and tells you, **not** to use the hammer because it is considered an **anti-pattern**.  
You ask him, *Why not a hammer?* The usual answer you receive will be something like,
*Oh, someone tried to put screws into the wall with a hammer, but failed and hit the thumb badly.* **Therefore all 
hammers must be replaced with screwdrivers!**  
It's quite obvious that the problem originates from choosing the wrong tool or using it in a wrong way.

## Service Container as Service Locator
Lately I came across Slim 4 because I wanted a no-nonsense lightweight framework. Since the interface is nice and easy 
to understand, I took a look into the implementation. The first thing making me shiver was the constructor of the App class.
Let's be clear here, It's about what the current state of the implementation looks like to me, not about blaming anyone.

```php
class App extends RouteCollectorProxy implements RequestHandlerInterface {
// omitted lines
    public function __construct(
        ResponseFactoryInterface $responseFactory,
        ?ContainerInterface $container = null,
        ?CallableResolverInterface $callableResolver = null,
        ?RouteCollectorInterface $routeCollector = null,
        ?RouteResolverInterface $routeResolver = null,
        ?MiddlewareDispatcherInterface $middlewareDispatcher = null
    ) {
        parent::__construct(
            $responseFactory,
            $callableResolver ?? new CallableResolver($container),
            $container,
            $routeCollector
        );

        $this->routeResolver = $routeResolver ?? new RouteResolver($this->routeCollector);
        $routeRunner = new RouteRunner($this->routeResolver, $this->routeCollector->getRouteParser(), $this);

        if (!$middlewareDispatcher) {
            $middlewareDispatcher = new MiddlewareDispatcher($routeRunner, $this->callableResolver, $container);
        } else {
            $middlewareDispatcher->seedMiddlewareStack($routeRunner);
        }

        $this->middlewareDispatcher = $middlewareDispatcher;
    }
// omitted lines
}
```

So there is one required dependency and five optional dependencies?
Digging deeper, I found nearly every _optional_ dependency assigned like this
`$dependency ?? new Dependency($otherDependency [, ...])`. So are those really optional? Obviously not.
The original idea of [Dependency Injection](https://www.martinfowler.com/articles/injection.html) is, that classes should not instantiate dependencies themselves.
I looked at the tests and had another disturbing encounter, `AppTest::testDoesNotUseContainerAsServiceLocator()`.
So Service Locator is considered an anti-pattern and therefore being desperately avoided, it seems the code has become much more complicated.

## Which problem is solved by a Design Pattern?
For me the Service Container solves a simple problem: Where should I create objects with dependencies?
While I agree on not injecting the container as a locator, on application level this makes little to no sense to me.
All those _not so optional_ dependencies can be provided by defaults in the container.
So I decided to refactor and re-add the container as a locator and use it in a proper way. Here is the result of the first iteration.  

```php
// container definitions
[
    App::class => function (Container $c) {
        return new App($c);
    },
    ResponseFactoryInterface::class => function () {
        return \Slim\Factory\AppFactory::determineResponseFactory();
    },
    CallableResolverInterface::class => function (Container $c) {
        return new \Slim\CallableResolver($c);
    },
    RouteCollectorInterface::class => function (Container $c) {
        return new RouteCollector(
            $c->get(ResponseFactoryInterface::class),
            $c->get(CallableResolverInterface::class),
            $c
        );
    },
    MiddlewareDispatcherInterface::class => function () {
        return new MiddlewareStack();
    },
    RouteResolverInterface::class => function (Container $c) {
        return new RouteResolver(
            $c->get(RouteCollectorInterface::class)
        );
    },
    \Psr\Http\Message\ServerRequestInterface::class =>
        ServerRequestCreatorFactory::create()->createServerRequestFromGlobals()
];
```

```php
class App extends RouteCollectorProxy implements RequestHandlerInterface {
// omitted lines
    public function __construct(Container $container) {
        parent::__construct(
            $container->get(ResponseFactoryInterface::class),
            $container->get(CallableResolver::class),
            $container,
            $container->get(RouteCollectorInterface::class)
        );
        $this->middlewareStack = $container->get(MiddlewareDispatcherInterface::class);
        $this->middlewareStack->seedMiddlewareStack(
            new RouteRunner($container->get(RouteResolverInterface::class), $this->routeCollector->getRouteParser(), $this)
        );
    }
// omitted lines
    public function addRoutingMiddleware(): RoutingMiddleware
    {
        $routingMiddleware = new RoutingMiddleware(
            $this->container->get(RouteResolverInterface::class),
            $this->container->get(RouteCollectorInterface::class)->getRouteParser()
        );

        $this->addMiddleware($routingMiddleware);

        return $routingMiddleware;
    }
// omitted lines
    public function addErrorMiddleware(
        bool $displayErrorDetails,
        bool $logErrors,
        bool $logErrorDetails
    ): ErrorMiddleware {
        $errorMiddleware = new ErrorMiddleware(
            $this->container->get(CallableResolver::class),
            $this->container->get(ResponseFactoryInterface::class),
            $displayErrorDetails,
            $logErrors,
            $logErrorDetails
        );
        $this->addMiddleware($errorMiddleware);

        return $errorMiddleware;
    }
// omitted lines
}
```

What I see now, is some stuff preventing me from removing the container:
- App extending _RouteCollectorProxy_  
- RouteRunner's dependency to _RouteCollectorProxy_, leading to a cyclic dependency  
- App's method _addRoutingMiddleware_  
- App's method _addErrorMiddleware_  
- ...

A lot of classes are poorly encapsulated. Often the encapsulation is broken, even by reflection for testing purposes. 
Most of the time the state of an object is tested but not really the behaviour. Which leads the to the following problem.

## The problem of writing code
We are exposed to poorly written code everyday.  
We believe code is hard to write because we see code that is hard to understand.  
We are obsessed with technology that magically solves the *problem of writing code*.  
We are failing, because we are left behind with nearly no skills except for reading the documentation of the next promising magical solution.  

**If you go to an expensive restaurant with a famous chef, would you be okay being served convenience food?**  

While thinking about this, let's take a look at the result of some serious refactoring.

## A few iterations of refactoring later...

```php
// container definitions
[
    App::class => function (Locator $kernel) {
        return new App(
            $kernel,
            $kernel->get(RouteCollector::class),
            new MiddlewareContainer($kernel->get(RouteRunner::class))
        );
    },
    InvocationStrategyInterface::class => new RequestResponse(),
    AdvancedCallableResolverInterface::class => function (Locator $kernel) {
        // Here we should pass a different Locator than the kernel
        return new \Slim\CallableResolver($kernel);
    },
    ResponseFactory::class => function () {
        return \Slim\Factory\AppFactory::determineResponseFactory();
    },
    CallableResolverInterface::class => function (Locator $kernel) {
        return new \Slim\CallableResolver($kernel);
    },
    Routing\RouteCollector::class => function (Locator $kernel) {
        return new Routing\FastRouter(
            $kernel,
            new FastRoute\RouteCollector(
                new FastRoute\RouteParser\Std(),
                new FastRoute\DataGenerator\GroupCountBased()
            )
        );
    },
    RoutingMiddleware::class => function (Locator $kernel) {
        return new RoutingMiddleware(
            $kernel->get(Routing\RouteCollector::class)
        );
    },
    ErrorMiddleware::class => function (Locator $kernel) {
        return new ErrorMiddleware(
            $kernel->get(CallableResolverInterface::class),
            $kernel->get(ResponseFactory::class),
            false,
            false,
            false
        );
    },
    ServerRequest::class => ServerRequestCreatorFactory::create()->createServerRequestFromGlobals(),
    ResponseEmitter::class => function () { return new Testing\ResponseEmitterStub(); },
];
```

```php
<?php

namespace Virtue\Api;

use Psr\Container\ContainerInterface as Locator;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as ServerRequest;
use Psr\Http\Server\RequestHandlerInterface as HandlesServerRequests;
use Slim\ResponseEmitter;
use Virtue\Api\Middleware\MiddlewareContainer;
use Virtue\Api\Routing\Api;
use Virtue\Api\Routing\RouteCollector;
use Virtue\Api\Routing\RouteGroup;

class App extends Api implements HandlesServerRequests
{
    /** @var string */
    public const VERSION = '0.0.0';
    /** @var MiddlewareContainer */
    protected $middlewares;

    public function __construct(
        Locator $kernel,
        RouteCollector $routeCollector,
        MiddlewareContainer $middlewares
    ) {
        parent::__construct($kernel, $routeCollector);
        $this->middlewares = $middlewares;
    }

    public function add(string $middleware): void
    {
        $this->middlewares->append($this->kernel->get($middleware));
    }

    public function run(?ServerRequest $request = null): void
    {
        $request = $request ?? $this->kernel->get(ServerRequest::class);

        $this->kernel->get(ResponseEmitter::class)->emit($this->handle($request));
    }

    public function handle(ServerRequest $request): Response
    {
        return $this->middlewares->handle($request);
    }
}
```

Yes, **that's all** - quite simple, isn't it? All the heavy work of creating application level dependencies went to the Service Container.
To add `RoutingMiddleware` to the `MiddlewareStack` I just do `$app->add(RoutingMiddleware::class)` and the `App` doesn't care about the dependencies of `RoutingMiddleware`, the `Locator` does.
Also when testing, I can replace the `ResponseEmitter` (which is a class, here an Interface would make sense) with a Stub implementation giving access to the last emitted `Response` for assertions.

```php
class ResponseEmitterStub extends ResponseEmitter
{
    /** @var Response */
    private $response;

    public function emit(Response $response): void
    {
        $this->response = $response;
    }

    public function last(): Response
    {
        return $this->response;
    }
}
```

And you know what is best? `AppTest::testDoesNotUseContainerAsServiceLocator()` is **still green**!

For more details look at the [commit history](https://github.com/vicephp/Virtue-Api/commits/master).

## Understand the Design Pattern and the problem it solves first
Design Patterns solve particular problems. It is important to understand the problem, in order to apply an appropriate 
Design Pattern. When a Design Pattern causes more problems than it solves, ask yourself, *Do I get the pattern right?*, *Do I understand the underlying problem?*
Way too often, a Design Pattern is prematurely applied, leading to mayhem and therefore branded as anti-pattern.
