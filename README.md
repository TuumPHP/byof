Build Your Own Framework
====================

1. Write a wish, first.
2. develop a framework. 
3. Then, rewrite the wish, and continue...

Overall Design
----------------

Ths document discusses about designs behind the web application framework development. The goal is to develop a web application framework that is capable to support for a very long term; say 5 to 10 years. 

#### [X] Design# Depend on Components

Framework changes as time goes by. But (hopefuly) not so for each components. Matured components that have a good focus should not change much. 

The framework should be a thin layer that glue these components. 

#### [X] Design# Loose Coupling

Components should be loosely coupled. 

Yet, it offers a nice magic like Laravel. Yeah, that sounds nice. 

#### [X] Design# Front End Framework (No infrastructures)

The web technology might evolve very quickly. Whereas the backend servers (and clients business) does not change that much. If they do, the change would require rewriting the code... 

So, the framework will focus on front end web server. No database stuff. 

Designing a Web Application Framework
------------------------------------------

So, what is web application? There are no single definition but I have a very simple one to start. It's by Symfony's; 

```php
$response = $app( $request );
```

and ```$app``` is the web application. 
The ```$request``` represents the incoming request, and the ```$response``` contains the response back to the client. 

All I have to do is, desing what features each of ```$app```, ```$request```, and ```$response``` have, and how to construct them. 

#### [X] Design# Immutable

I heard a lot of good thing about being immutable. 

So, let's say, ```$app``` should be immutable. 

That means, all the values that varies are stored in ```$request```, ```$response```, or outside of $app as a resource. 

> I know asking for immutable, or multi-thread-ready is too much for PHP, but I hope this restriction will result in cleaner and simpler framework design. 


Chain of Responsibility for HTTP Stack
-----------

#### [X] Design# Use Middlewares

[StackPHP](http://stackphp.com/) gives a good design for a web application framework. There is [another excellent article about use of middlewares](https://mwop.net/blog/2015-01-08-on-http-middleware-and-psr-7.html). Using middleware should be the way to go. 

#### [X] Design# Everything is a Middleware

Slim and Silex also uses middleware design. But the last middleware is always the $app itself, which has routing, events, all kind of stuff. 

Well, there should be an examples, such as DI container, logger, etc. 

### Middleware Stack Design

But there are one thing I like to complain. The middleware have too many responsibilities, such as,

*   do whatever as a middleware, and
*   call next middleware. 

Yes, there are 2 of them. So, I like to separate the responsibility to different objects.

to-be-written.


Request and Reponse
-------------------

#### [X] Design# Use Symfony's Http-Foundation

At the time of this writing, the PSR-7 is still under development. So, for now, I will use Symfony's Http-Foundation as request and response. 

But HttpKernelInterface uses the ```Request``` class as a typehint, which should be using an interface (the upcoming PSR-7 interface, that is). This make things very difficult to design something. So, I will not adopt the HttpKernelInterface.

### Request Class

Extend Symfony's ```Request``` class. 

#### [_] Design# Request maybe is Immutable

The request, for most of the part, is immutable, except for sessions and attributes, which can be altered during the process.  Though, it's probably possible to think that, 

*   session is a resource, and 
*   the response keeps the attributes.

### Response Class

Extend Symfony's ```Response``` class. 

#### [X] Design# Generate Response From Request

$request is the input to the web application. Some where along the middleware stack, a response must be created. 

To simplify the creation of $response, I like to introduce a factory object into the $request as follows.

A View response. 

```php
$request->respond()->view('welcome')->with('message', 'hello');
```

or, Redirect response. 

```php
$request->redirect()->to('login')->withError('failed to login');
```

also, Error response. 

```php
$request->error()->notFound()->withError('You cannot do that');
```

The redirection often requires ```$request``` information, such as baseURL, to redirect with relative to its url location. Creating a response directly from $request seems to be a nice trick to handle the $request data to $response object. 




Configuring the Application
------------------------------

### Directory Structure

```
project-root
  |- app/
  |- docs/
  |- public/
  |- src/
  |- var/
  |- view/
  +- vendor/
```

nothing special.

*   app: for storing configuration, scripts, etc.
*   docs: static files.
*   public: where .htaccess sits for serving.
*   src: class file for the project.
*   var: logs, caches, etc.
*   view: template files.
*   vendor: composer's territory. 

The view directory (and maybe the vendor directory) probably are not source controlled. It should be possible to configure the location of directories, such as docs, view, and var. 

### Union File Manager

The ```$app``` construction must reflect the environment, such as production, local, staging, etc. use the "union file manager" thing. A lot of framework uses it but I cannot find the name of the technique. Nothing special here, as well. 

An envrionment specific directory structure (such as testing) can The original file system You can override the file by specifying 

#### [X] Design# No DI Container Component!

The Union File Manager controls the construction process, i.e. it is the DI container. No external DI container, such as Pimple, are used. In other words, I will create my own simple container. 

Once the DI process has a standard, or de-fact standard arises, I would like to use it, though. 

#### [X] Design# Factory Method: Forge

Use static ```forge``` method as standard factory method for various classes. Ideally, the factory method will not need any arguments. 


Router and Controller
---------------------

### Router

PHP already have tons of routers to choose from. So, I like to introduce a very simple interface for the adaptors to these routers. 

```php
interface RouterInterface {
    public function match($request);
    public function getRouting();
    public function getReverseRoute($request);
}

interface ReverseRouteInterface {
    public function generate($name, $args=[]);
}
```

*   ```RouterInterface::match```: matches the route against $request.
*   ```RouterInterface::getRouting ```: returns the object to set routes. The returned object varies from router to router. 
*   ```RouterInterface::getReverseRoute ```: returns a RouteNamesInterface object to produce a url by a route name.
*   ```ReverseRouteInterface::generate```: generate a route for a given ```$name``` with parameter, ```$args```.

#### [_] Design# Separate Dispatcher 

To accomodate other strategies for dispatching a controller, separate the dispatching from the main stack. 


### Default Dispatching Strategy

The default behavior of the dispatcher is, the controller is also another ```StackHandleInterface``` object.

So, specify one of the followings as the dispatchable controller. 

*   An object implementing ```StackHandleInterface``` (i.e. closure with $request as an argument).
*   A controller class name which implements the StackHandleInterface. 

To represent these, the router must return only, 

*   controller: a class name, or a StackHandleInterface closure. 
*   parameters: an array of values from the router. 

additinaly,

*   before filter,
*   after filter, 

#### [_] Design# Dispatching Inside Controller 

Inside the controller, it can dispatch other methods based on the request. This is a nice feature because controller is the best person who knows how to handle the request, a resource, restful, or whatever. 

####[_] Design# Use Trait for Controllers

Some ideas for dispatching inside controllers. 

A simple dispatcher is to use method name. The arguments maybe fulfilled based on the analyzed route and $_REQUEST parameters. Example: 

```php
class MyController {
  use ControllerByMethodTrait;
  protected on{$Method}($id,...) {}
}
```

A simple reg-ex routing is used to dispatch a method. 

```php
class MyController {
  use ControllerByRouteTrait;
  protected $routes = [
      'createForm'   => 'get:/create',
      'getSomething' => 'get:/(id:[0-9]+)',
      'postData'     => 'post:/create',
  ];
  protected getSomething($id) {}
}
```

How about a resource type dispatcher?

```php
class MyController {
  use ControllerResourceTrait;
  protected onGet($id) {}
}
```


#### [_] find a simple and fast router for the route based dispatcher.. 

I am looking for a router implementation that is small, simple, clean, but very fast for a small set of routes. 


Stacks
------

Lots of ideas and code from [no framework tutorial](https://github.com/PatrickLouys/no-framework-tutorial) in GitHub. 

### Error Stack

Takes care of the errors. 

ErrorHandle is at the very beginning of the stack. 
Set up [filp/whoops](https://github.com/filp/whoops) in debug mode. Set an error-viewer (to be desgined) if not. 

ErrorRelease sits at the end of the stack, converting an error response into an error page. to be designed. 


### Session Stack

Get/set data to session. 

SessionHandle takes flash data into the ```$response```, using $request's setAttribute method. This way, the flashed data will always passed to response. If the response was another redirect, it defaults to pass the flash data to the next request. 

SessionRelease retrieves data from Redirect responses into session's flash. If the response is a View response, it retrieves the C.S.R.F. token value and set it to the session. 


### View Stack

For View response, converts the template file using template engine. 


View and Template
-----------------

Default template is PHP; It is a lot easier to debug PHP file. And there are already many PHP based template components, such as [League's Plates](http://platesphp.com/) and [Aura.View](https://github.com/auraphp/Aura.View). 

I like to introduce a super simple interface for these 

```php
interface ViewEngineInterface {
    public function render($file, $data = []);
}
```

#### [_] Design# View Must Be Immutable

The view template often uses URL generator, which uses $request. The view template must be designed to be immutable. 

The rendering must happen in an object. 

