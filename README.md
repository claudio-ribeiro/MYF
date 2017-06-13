*Symfony is a reusable set of standalone, decoupled and cohesive PHP components that solve common web development problems.*

Instead of creating a framework from scratch, we will write the same application over and over again, adding layers of complexity as we go. We're starting with the simplest PHP application we can think of:

```PHP
// framework/index.php
$input = $_GET['name'];

printf('Hello %s', $input);
```

Then we can use the PHP in built server to test this code:

```bash
php -S 127.0.0.1:4321
```

 `http://localhost:4321/index.php?name=Claudio`, check it on the browser.

#### The HTTPFoundation component

The HTTP specification describes how a client (a browser for instance) interacts with a server (our application via a web server). The dialog between the client and the server is specified by well-defined messages, requests and responses: the client sends a request to the server and based on this request, the server returns a response.

In PHP, the request is represented by global variables ($_GET, $_POST, $_FILE, $_COOKIE, $_SESSION...) and the response is generated by functions (echo, header, setcookie, ...).

The first step towards better code is probably to use an Object-Oriented approach; that's the main goal of the Symfony HttpFoundation component: replacing the default PHP global variables and functions by an Object-Oriented layer.

**HTTPFoundation: defines an object oriented layer for the HTTP specification.**

```bash
composer require symfony/http-foundation
```

We are now able to rewrite our application using the `Request` and `Response` classes.

```php
// framework/index.php
require_once __DIR__.'/vendor/autoload.php';

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

$request = Request::createFromGlobals();

$input = $request->get('name', 'World');

$response = new Response(sprintf('Hello %s', htmlspecialchars($input, ENT_QUOTES, 'UTF-8')));

$response->send();
```
#### Front Controller
What happens when we want to add a new page? We're going to start having repeated code that we'll need to separate. For that we create a single point of entry: a front controller.

*Exposing a single PHP script to the end user is a design pattern called the 'front controller'.*

```php
// framework/front.php
require_once __DIR__.'/vendor/autoload.php';

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

$request = Request::createFromGlobals();
$response = new Response();

$map = array(
    '/hello' => __DIR__.'/hello.php',
    '/bye'   => __DIR__.'/bye.php',
);

$path = $request->getPathInfo();
if (isset($map[$path])) {
    require $map[$path];
} else {
    $response->setStatusCode(404);
    $response->setContent('Not Found');
}

$response->send();
```
and the hello.php script:

```php
// framework/hello.php
$input = $request->get('name', 'World');
$response->setContent(sprintf('Hello %s', htmlspecialchars($input, ENT_QUOTES, 'UTF-8')));
```

Using the in built PHP server just as earlier we can acess the following url's:

`127.0.0.1:4321 -t web/ web/indexphp`

127.0.0.1:4321/front.php/hello?name=Claudio

127.0.0.1:4321/front.php/bye

#### Routing

One very important aspect of any website is the form of its URLs. Thanks to the URL map, we have decoupled the URL from the code that generates the associated response, but it is not yet flexible enough. For instance, we might want to support dynamic paths to allow embedding data directly into the URL (e.g. /hello/Claudio) instead of relying on a query string (e.g. /hello?name=Claudio).

To support this feature, add the Symfony Routing component as a dependency:

```bash
composer require symfony/routing
```

**Routing: Maps an HTTP request to a set of of configuration variables.**

```php
// example.com/web/front.php
require_once __DIR__.'/../vendor/autoload.php';

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing;

$request = Request::createFromGlobals();
$routes = include __DIR__.'/../src/app.php';

$context = new Routing\RequestContext();
$context->fromRequest($request);
$matcher = new Routing\Matcher\UrlMatcher($routes, $context);

try {
    extract($matcher->match($request->getPathInfo()), EXTR_SKIP);
    ob_start();
    include sprintf(__DIR__.'/../src/pages/%s.php', $_route);

    $response = new Response(ob_get_clean());
} catch (Routing\Exception\ResourceNotFoundException $e) {
    $response = new Response('Not Found', 404);
} catch (Exception $e) {
    $response = new Response('An error occurred', 500);
}

$response->send();
```

Our hello.php template:

```php
Hello <?php echo htmlspecialchars($name, ENT_QUOTES, 'UTF-8') ?>
```

and finnaly our app.php, which contains all of our routes now:

```php
use Symfony\Component\Routing;

$routes = new Routing\RouteCollection();
$routes->add('hello', new Routing\Route('/hello/{name}', array('name' => 'World')));
$routes->add('bye', new Routing\Route('/bye'));

return $routes;
```

#### Templating

Our framework hardcodes the way specific "code" (the templates) is run. For simple pages like the ones we have created so far, that's not a problem, but if you want to add more logic, you would be forced to put the logic into the template itself, which is probably not a good idea, especially if you still have the separation of concerns principle in mind.

Let's separate the template code from the logic by adding a new layer: **the controller: The controller's mission is to generate a Response based on the information conveyed by the client's Request.**

Here's our updated version:

```php
require_once __DIR__.'/../vendor/autoload.php';

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing;

function render_template($request)
{
    extract($request->attributes->all(), EXTR_SKIP);
    ob_start();
    include sprintf(__DIR__.'/../src/pages/%s.php', $_route);

    return new Response(ob_get_clean());
}

$request = Request::createFromGlobals();
$routes = include __DIR__.'/../src/app.php';

$context = new Routing\RequestContext();
$context->fromRequest($request);
$matcher = new Routing\Matcher\UrlMatcher($routes, $context);

try {
    $request->attributes->add($matcher->match($request->getPathInfo()));
    $response = call_user_func($request->attributes->get('_controller'), $request);
} catch (Routing\Exception\ResourceNotFoundException $e) {
    $response = new Response('Not Found', 404);
} catch (Exception $e) {
    $response = new Response('An error occurred', 500);
}

$response->send();
```

At this point we can create a new application, just by modifying our `app.php` file. That's what we will do, create a new application, just by modifying this single file.

```php
use Symfony\Component\Routing;
use Symfony\Component\HttpFoundation\Response;

function is_leap_year($year = null) {
    if (null === $year) {
        $year = date('Y');
    }

    return 0 === $year % 400 || (0 === $year % 4 && 0 !== $year % 100);
}

$routes = new Routing\RouteCollection();
$routes->add('leap_year', new Routing\Route('/is_leap_year/{year}', array(
    'year' => null,
    '_controller' => function ($request) {
        if (is_leap_year($request->attributes->get('year'))) {
            return new Response('Yep, this is a leap year!');
        }

        return new Response('Nope, this is not a leap year.');
    }
)));

return $routes;
```
#### The HTTPKernel component

Right now, all our examples use procedural code, but remember that controllers can be any valid PHP callbacks. Let's convert our controller to a proper class:

```php
class LeapYearController
{
    public function indexAction($request)
    {
        if (is_leap_year($request->attributes->get('year'))) {
            return new Response('Yep, this is a leap year!');
        }

        return new Response('Nope, this is not a leap year.');
    }
}
```
And of course, we need to update the route definition:

```php
$routes->add('leap_year', new Routing\Route('/is_leap_year/{year}', array(
    'year' => null,
    '_controller' => array(new LeapYearController(), 'indexAction'),
)));
```

The move is pretty straightforward and makes a lot of sense as soon as you create more pages but you might have noticed a non-desirable side-effect... The LeapYearController class is always instantiated, even if the requested URL does not match the leap_year route. This is bad for one main reason: performance wise, all controllers for all routes must now be instantiated for every request. It would be better if controllers were lazy-loaded so that only the controller associated with the matched route is instantiated.

To solve this issue, and a bunch more, let's install and use the HttpKernel component:

 `composer require symfony/http-kernel`

The HttpKernel component has many interesting features, but the ones we need right now are the controller resolver and argument resolver. A controller resolver knows how to determine the controller to execute and the argument resolver determines the arguments to pass to it, based on a Request object. 

With this, we can just inject the `$year` request attribute for our controller:

```php
class LeapYearController
{
    public function indexAction($year)
    {
        if (is_leap_year($year)) {
            return new Response('Yep, this is a leap year!');
        }

        return new Response('Nope, this is not a leap year.');
    }
}
```
And we have the newest version of our framework:

```php
require_once __DIR__.'/../vendor/autoload.php';

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing;
use Symfony\Component\HttpKernel;

function render_template(Request $request)
{
    extract($request->attributes->all(), EXTR_SKIP);
    ob_start();
    include sprintf(__DIR__.'/../src/pages/%s.php', $_route);

    return new Response(ob_get_clean());
}

$request = Request::createFromGlobals();
$routes = include __DIR__.'/../src/app.php';

$context = new Routing\RequestContext();
$context->fromRequest($request);
$matcher = new Routing\Matcher\UrlMatcher($routes, $context);

$controllerResolver = new HttpKernel\Controller\ControllerResolver();
$argumentResolver = new HttpKernel\Controller\ArgumentResolver();

try {
    $request->attributes->add($matcher->match($request->getPathInfo()));

    $controller = $controllerResolver->getController($request);
    $arguments = $argumentResolver->getArguments($request, $controller);

    $response = call_user_func_array($controller, $arguments);
} catch (Routing\Exception\ResourceNotFoundException $e) {
    $response = new Response('Not Found', 404);
} catch (Exception $e) {
    $response = new Response('An error occurred', 500);
}
```

#### Separation of concerns

One down-side of our framework right now is that we need to copy and paste the code in front.php each time we create a new website. 60 lines of code is not that much, but it would be nice if we could wrap this code into a proper class. It would bring us better reusability and easier testing to name just a few benefits.

If you have a closer look at the code, front.php has one input, the Request and one output, the Response. Our framework class will follow this simple principle: the logic is about creating the Response associated with a Request.

Let's create our very own namespace for our framework: Simplex. Move the request handling logic into its own Simplex\\Framework class:

```php
namespace Simplex;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Controller\ArgumentResolver;
use Symfony\Component\HttpKernel\Controller\ControllerResolver;
use Symfony\Component\Routing\Exception\ResourceNotFoundException;
use Symfony\Component\Routing\Matcher\UrlMatcher;

class Framework
{
    protected $matcher;
    protected $controllerResolver;
    protected $argumentResolver;

    public function __construct(UrlMatcher $matcher, ControllerResolver $controllerResolver, ArgumentResolver $argumentResolver)
    {
        $this->matcher = $matcher;
        $this->controllerResolver = $controllerResolver;
        $this->argumentResolver = $argumentResolver;
    }

    public function handle(Request $request)
    {
        $this->matcher->getContext()->fromRequest($request);

        try {
            $request->attributes->add($this->matcher->match($request->getPathInfo()));

            $controller = $this->controllerResolver->getController($request);
            $arguments = $this->argumentResolver->getArguments($request, $controller);

            return call_user_func_array($controller, $arguments);
        } catch (ResourceNotFoundException $e) {
            return new Response('Not Found', 404);
        } catch (\Exception $e) {
            return new Response('An error occurred', 500);
        }
    }
}
```

And update the index.php file:

```php
$request = Request::createFromGlobals();
$routes = include __DIR__.'/../src/app.php';

$context = new Routing\RequestContext();
$matcher = new Routing\Matcher\UrlMatcher($routes, $context);

$controllerResolver = new ControllerResolver();
$argumentResolver = new ArgumentResolver();

$framework = new Simplex\Framework($matcher, $controllerResolver, $argumentResolver);
$response = $framework->handle($request);

$response->send();
```

For the classes defined under the Simplex and Calendar namespaces to be autoloaded, update the composer.json file:

```json
{
    "...": "...",
    "autoload": {
        "psr-4": { "": "src/" }
    }
}
```

After this, create the `src/Calendar/Controller/LeapYearController.php` and `src/Calendar/Model/LeapYear.php` classes and update the route. This way we obtain a bigger separation of concerns.

 
That's it! Our application has now four different layers and each of them has a well defined goal:

**web/front.php:** The front controller; the only exposed PHP code that makes the interface with the client (it gets the Request and sends the Response) and provides the boiler-plate code to initialize the framework and our application;

**src/Simplex:** The reusable framework code that abstracts the handling of incoming Requests (by the way, it makes your controllers/templates easily testable -- more about that later on);

**src/Calendar:** Our application specific code (the controllers and the model);

**src/app.php:** The application configuration/framework customization.

#### Unit testing
The next step is all about adding unit testing to the framework using phpunit. Nothing new here, check the code if you want to have an idea. Move along ;)


#### The Event Dispatcher component

Our framework is still missing a major characteristic of any good framework: extensibility. Being extensible means that the developer should be able to easily hook into the framework life cycle to modify the way the request is handled.

What kind of hooks are we talking about? Authentication or caching for instance. To be flexible, hooks must be plug-and-play; the ones you "register" for an application are different from the next one depending on your specific needs.

As there is no standard for PHP, we are going to use a well-known design pattern, the Mediator, to allow any kind of behaviors to be attached to our framework; the Symfony EventDispatcher Component implements a lightweight version of this pattern:

 `composer require symfony/event-dispatcher`

How does it work? The dispatcher, the central object of the event dispatcher system, notifies listeners of an event dispatched to it. Put another way: your code dispatches an event to the dispatcher, the dispatcher notifies all registered listeners for the event, and each listener do whatever it wants with the event.

#### HttpKernel Component: HttpKernelInterface

One of the main goals of this development is to achieve interoperability between frameworks and applications using Symfony components. A big step towards this goal is to make our framewirk implement HttpKernelInterface. HttpKernelInterface is probably the most important piece of code in the HttpKernel component, no kidding. Frameworks and applications that implement this interface are fully interoperable. 

Let's update our framework to implement it:

```php
// ...
use Symfony\Component\HttpKernel\HttpKernelInterface;

class Framework implements HttpKernelInterface
{
    // ...

    public function handle(
        Request $request,
        $type = HttpKernelInterface::MASTER_REQUEST,
        $catch = true
    ) {
        // ...
    }
}
```

This simple change brings us a lot of benefits. One of the most impressive ones is *transparent HTTP caching support*.

The HttpCache class implements a fully-featured reverse proxy, written in PHP; it implements HttpKernelInterface and wraps another HttpKernelInterface instance:

```php
// example.com/web/front.php
$framework = new Simplex\Framework($dispatcher, $matcher, $resolver);
$framework = new HttpKernel\HttpCache\HttpCache(
    $framework,
    new HttpKernel\HttpCache\Store(__DIR__.'/../cache')
);

$framework->handle($request)->send();
```

That's all it takes to add HTTP caching support to our framework. 

Configuring the cache needs to be done via HTTP cache headers. For instance, to cache a response for 10 seconds, use the Response::setTtl() method:

```php
// ...
public function indexAction(Request $request, $year)
{
    $leapyear = new LeapYear();
    if ($leapyear->isLeapYear($year)) {
        $response = new Response('Yep, this is a leap year!');
    } else {
        $response = new Response('Nope, this is not a leap year.');
    }

    $response->setTtl(10);

    return $response;
}
```
Using HTTP cache headers to manage your application cache is very powerful and allows you to tune finely your caching strategy as you can use both the expiration and the validation models of the HTTP specification. If you are not comfortable with these concepts, read the HTTP caching chapter of the Symfony documentation.

The Response class contains many other methods that let you configure the HTTP cache very easily. One of the most powerful is setCache() as it abstracts the most frequently used caching strategies into one simple array:

```php
$date = date_create_from_format('Y-m-d H:i:s', '2005-10-15 10:00:00');

$response->setCache(array(
    'public'        => true,
    'etag'          => 'abcde',
    'last_modified' => $date,
    'max_age'       => 10,
    's_maxage'      => 10,
));

// it is equivalent to the following code
$response->setPublic();
$response->setEtag('abcde');
$response->setLastModified($date);
$response->setMaxAge(10);
$response->setSharedMaxAge(10);
```

#### HTTPKernel Component: The HTTPKernel class

If you were to use our framework right now, you would probably have to add support for custom error messages. We do have 404 and 500 error support but the responses are hardcoded in the framework itself. Making them customizable is easy enough though: dispatch a new event and listen to it. Doing it right means that the listener has to call a regular controller.

Enter the HttpKernel class. Instead of solving the same problem over and over again and instead of reinventing the wheel each time, the HttpKernel class is a generic, extensible and flexible implementation of HttpKernelInterface.

This class is very similar to the framework class we have written so far: it dispatches events at some strategic points during the handling of the request, it uses a controller resolver to choose the controller to dispatch the request to, and as an added bonus, it takes care of edge cases and provides great feedback when a problem arises.

Sources:

https://www.sitepoint.com/build-php-framework-symfony-components/

https://symfony.com/doc/current/create_framework/routing.html
