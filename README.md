Build Your Own Framework
====================

Write a wish, first.

Then, I develop a framework. 

Then, rewrite the wish (continue).

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
------------------------

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

To simplify the creation of $response, I like to introduce a factory object into the $request, such as 

```php
$request->respond()->view('welcome')->with('message', 'hello');
```

Or, 

```php
$request->redirect()->to('login')->withError('failed to login');
```

The redirection often requires $request information, such as baseURL, to redirect with relative to its url location. Creating a response directly from $request seems to be a nice trick to handle the $request data to $response object. 




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
-------------------------

### Routers

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
*   ```RouterInterface::getReverseRoute ```: returns a reverse route by name RouteNamesInterface object.
*   ```ReverseRouteInterface::generate```: generate a route for a given ```$name``` with parameter, ```$args```.


### Dispatching and Controller

The followings are the dispatchables. 

*   StackHandleInterface (i.e. closure with $request as an argument).
*   A controller class name which implements the StackHandleInterface. 
*   controller and a method name to invoke.  

To represent these, the router must return the following. 

*   controller: a class name, or a StackHandleInterface closure. 
*   method: a method name if present. 

### Dispatching Inside Controller 

A controller implementing StackHandleInterface may have only one public interface, ```__invoke($request)```. Inside the handle, it is possible, or useful, to dispatch to its own method. 

#### Design# [_] Dispatch based on method

A simple dispatcher is to use method name. The arguments maybe fulfilled based on the analyzed route and $_REQUEST parameters. Example: 

```php
class MyController {
  use MethodDispatcherTrait;
  protected on{$Method}($id,...) {}
}
```

#### Design# [_] Dispatch based on local routing information

A simple reg-ex routing is used to dispatch a method. 

```php
class MyController {
  use RouteDispatcherTrait;
  protected getRoutes() {
    return [
      'createForm' => 'get:/create',
      'getSomething' => 'get:/(id:[0-9]+)',
      'postData' => 'post:/create',
    ];
  }
  protected getSomething($id) {}
}
```

#### [_] find a simple and fast router for the route based dispatcher.. 



View and Template
---------------------

I like to introduce a super simple interface for these 

```php
interface ViewEngineInterface {
    public function render($file, $data = []);
}
```