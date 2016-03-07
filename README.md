# Blast Turbine

[![Latest Version on Packagist][ico-version]][link-packagist]
[![Software License][ico-license]](LICENSE.md)
[![Build Status][ico-travis]][link-travis]
[![Total Downloads][ico-downloads]][link-downloads]
[![Coverage Status][ico-coveralls]][link-coveralls]

Turbine is a advanced derivate of [Proton](https://github.com/alexbilbie/Proton) and is a [PSR-7](https://github.com/php-fig/http-message) and [StackPHP](http://stackphp.com/) compatible micro framework.

Turbine uses latest versions of [League\Route](https://github.com/thephpleague/route) for routing, [League\Container](https://github.com/thephpleague/container) for dependency injection, and [League\Event](https://github.com/thephpleague/event) for event dispatching.

## Installation

Just add `"blast/turbine": "~1.0"` to your `composer.json` file.

## Setup

Basic usage with anonymous functions:

```php
// index.php
<?php

require __DIR__.'/../vendor/autoload.php';

$app = new Turbine\Application();

$app->get('/', function ($request, $response) {
    $response->getBody()->write('<h1>It works!</h1>');
    return $response;
});

$app->get('/hello/{name}', function ($request, $response, $args) {
    $response->getBody()->write(
        sprintf('<h1>Hello, %s!</h1>', $args['name'])
    );
    return $response;
});

$app->run();
```

Basic usage with controllers:

```php
// index.php
<?php

require __DIR__.'/../vendor/autoload.php';

$app = new Turbine\Application();

$app->get('/', 'HomeController::index'); // calls index method on HomeController class

$app->run();
```

```php
// HomeController.php
<?php

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;

class HomeController
{
    public function index(ServerRequestInterface $request, ResponseInterface $response, array $args)
    {
        $response->getBody()->write('<h1>It works!</h1>');
        return $response;
    }
}
```

Constructor injections of controllers

```php
// index.php
<?php

require __DIR__.'/../vendor/autoload.php';

$app = new Turbine\Application();

$app->share('App\CustomService', new App\CustomService)
$app->get('/', 'HomeController::index'); // calls index method on HomeController class

$app->run();
```

```php
// HomeController.php
<?php

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;

class HomeController
{
    /**
     * @var App\CustomService
     */
    private $service;

    /**
     * @param App\CustomService $application
     */
    public function __construct(App\CustomService $service = null)
    {
        $this->service = $service;
    }

    /**
     * @return App\CustomService
     */
    public function getService()
    {
        return $this->service;
    }
    
    public function index(ServerRequestInterface $request, ResponseInterface $response, array $args)
    {
        //do somehing with service
        $service = $this->getService();
    
        $response->getBody()->write();
        return $response;
    }
}
```

Basic usage with StackPHP (using `Stack\Builder` and `Stack\Run`):

```php
// index.php
<?php
require __DIR__.'/../vendor/autoload.php';

$app = new Turbine\Application();

$app->get('/', function ($request, $response) {
    $response->setContent('<h1>It works!</h1>');
    return $response;
});

$httpKernel = new Turbine\Symfony\HttpKernelAdapter($app);

$stack = (new Stack\Builder())
    ->push('Some/MiddleWare') // This will execute first
    ->push('Some/MiddleWare') // This will execute second
    ->push('Some/MiddleWare'); // This will execute third

$app = $stack->resolve($httpKernel);
Stack\run($httpKernel); // The app will run after all the middlewares have run
```

### Routes from configuration

You can add routes from configuration  

```php
//configure
$app->setConfig('routes', function ($router, $app) {
    $router->get('/blog/', 'App\BlogController::getIndex');
});
```

## Debugging

By default Turbine runs with debug options disabled. To enable debugging add

```php
$app->setConfig('debug', true);
```

Turbine has built in support for Monolog. To access a channel call:

```php
$app->getLogger('channel name');
```

For more information about channels read this guide - [https://github.com/Seldaek/monolog/blob/master/doc/usage.md#leveraging-channels](https://github.com/Seldaek/monolog/blob/master/doc/usage.md#leveraging-channels).

## Custom exception decoration

```php
$app->setExceptionDecorator(function (\Exception $e) {
    return new TextResponse('Fail', 500);
});
```

## Events

You can intercept requests and responses at three points during the lifecycle:

### request.received

```php
$app->subscribe('request.received', function ($event) {
    // access the request using $event->getRequest()
})
```

This event is fired when a request is received but before it has been processed by the router.

### response.created

```php
$app->subscribe('response.created', function ($event) {
    // access the request using $event->getRequest()
    // access the response using $event->getResponse()
})
```

This event is fired when a response has been created but before it has been output.

### response.sent

```php
$app->subscribe('response.sent', function ($event) {
    // access the request using $event->getRequest()
    // access the response using $event->getResponse()
})
```

This event is fired when a response has been output and before the application lifecycle is completed.

### Custom Events

You can fire custom events using the event emitter directly:

```php
// Subscribe
$app->subscribe('custom.event', function ($event, $time) {
    return 'the time is '.$time;
});

// Publish
$app->getEventEmitter()->emit('custom.event', time());
```

### Events from configuration

You can add configured events from configuration  

```php
//configure
$app->setConfig('events', function ($emitter, $app) {
    $emitter->addListener('custom.event', function ($event, $time) {
        return 'the time is '.$time;
    });
});
```

## Dependency Injection Container

Turbine uses `League/Container` as its dependency injection container.

You can bind singleton objects into the container from the main application object using ArrayAccess:

```php
$app['db'] = function () {
    $manager = new Illuminate\Database\Capsule\Manager;

    $manager->addConnection([
        'driver'    => 'mysql',
        'host'      => $config['db_host'],
        'database'  => $config['db_name'],
        'username'  => $config['db_user'],
        'password'  => $config['db_pass'],
        'charset'   => 'utf8',
        'collation' => 'utf8_unicode_ci'
    ], 'default');

    $manager->setAsGlobal();

    return $manager;
};
```

or by accessing the container directly:

```php
$app->getContainer()->share('db', function () {
    $manager = new Illuminate\Database\Capsule\Manager;

    $manager->addConnection([
        'driver'    => 'mysql',
        'host'      => $config['db_host'],
        'database'  => $config['db_name'],
        'username'  => $config['db_user'],
        'password'  => $config['db_pass'],
        'charset'   => 'utf8',
        'collation' => 'utf8_unicode_ci'
    ], 'default');

    $manager->setAsGlobal();

    return $manager;
});
```

Multitons can be added using the `add` method on the container:

```php
//callback
$app->getContainer()->add('foo', function () {
    return new Foo();
});
```

Service providers can be registered using the `register` method on the Turbine app or `addServiceProvider` on the container:

```php
$app->register('\My\Service\Provider');
$app->getContainer()->addServiceProvider('\My\Service\Provider');
```

For more information about service providers check out this page - [http://container.thephpleague.com/service-providers/](http://container.thephpleague.com/service-providers/).

For easy testing down the road it is recommended you embrace constructor injection:

```php
$app->getContainer()->add('Bar', function () {
        return new Bar();
});

$app->getContainer()->add('Foo', function () use ($app) {
        return new Foo(
            $app->getContainer()->get('Bar')
        );
});
```

### Services from configuration

You can add service from configuration  

```php
//configure
$app->setConfig('services', function ($container, $app) {
    $container->add('foo', function () {
        return new Foo();
    });;
});
```

## Change log

Please see [CHANGELOG](CHANGELOG.md) for more information what has changed recently.

## Testing

``` bash
$ composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security

If you discover any security related issues, please email <mjls@web.de> instead of using the issue tracker.

## Credits

- [Marco Bunge](https://github.com/mbunge)
- [Alex Bilbie](https://github.com/alexbilbie) (Proton)
- [All contributors](https://github.com/phpthinktank/Turbine/graphs/contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

[ico-version]: https://img.shields.io/packagist/v/blast/turbine.svg?style=flat-square
[ico-license]: https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square
[ico-travis]: https://img.shields.io/travis/phpthinktank/blast-turbine/master.svg?style=flat-square
[ico-downloads]: https://img.shields.io/packagist/dt/blast/turbine.svg?style=flat-square
[ico-coveralls]: https://img.shields.io/coveralls/phpthinktank/blast-turbine/master.svg?style=flat-square)](https://coveralls.io/github/phpthinktank/blast-turbine?branch=1.0.x-dev

[link-packagist]: https://packagist.org/packages/blast/turbine
[link-travis]: https://travis-ci.org/phpthinktank/blast-turbine
[link-downloads]: https://packagist.org/packages/blast/turbine
[link-author]: https://github.com/mbunge
[link-contributors]: ../../contributors
[link-coveralls]: https://coveralls.io/github/phpthinktank/blast-turbine
