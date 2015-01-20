Build Your Own Framework
====================

Write a wish, first.

Then, I develop a framework. 

Then, rewrite the wish (continue).

Overall Design
----------------

Ths document discusses about designs behind the web application framework development. The goal is to develop a web application framework that is capable to support for a very long term; say 5 to 10 years. 

#### Design# Depend on Components

Framework changes as time goes by. But (hopefuly) not so for each components. Matured components that have a good focus should not change much. 

The framework should be a thin layer that glue these components. 

#### Design# Loose Coupling

Components should be loosely coupled. 

Yet, it offers a nice magic like Laravel. Yeah, that sounds nice. 

#### Design# Front End Framework (No infrastructures)

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

#### Design# Be Immutable

I heard a lot of good thing about being immutable. 

So, let's say, ```$app``` should be immutable. 

That means, all the values that varies are stored in ```$request```, ```$response```, or outside of $app as a resource. 

> I know asking for immutable, or multi-thread-ready is too much for PHP, but I hope this restriction will result in cleaner and simpler framework design. 


Starting from StackPHP
-----------

#### Design# Use Middlewares

[StackPHP](http://stackphp.com/) gives a good design for a web application framework. There is [another excellent article about use of middlewares](https://mwop.net/blog/2015-01-08-on-http-middleware-and-psr-7.html). Using middleware should be the way to go. 

#### Design# Everything is a Middleware

Slim and Silex also uses middleware design. But the last middleware is always the $app itself, which has routing, events, all kind of stuff. 

Well, there should be an examples, such as DI container, logger, etc. 

#### Modify Middleware Design

But there are one thing I like to complain. The middleware have too many responsibilities, such as,

*   do whatever as a middleware, and
*   call next middleware. 

Yes, there are 2 of them. So, I like to separate the responsibility to different objects.


Request and Reponse
------------------------

### Design# Use Symfony's Http-Foundation

Unfortunately, the PSR-7 is still under development. So, for now, I will use Symfony's Http-Foundation as request and response. 

But HttpKernelInterface uses class as a typehint, not an interface. This make things very difficult to design something. So, I will not adopt the HttpKernelInterface.

### Design# Create Response From Request

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

### Design# Request maybe Immutable

It can be, I guess. 

But current Symfony's Request class has attribute property which can contain various data. In the future, I would like to make the $request immutable, and store all the variables into $response object (inside the $request object, though). 


Configuring the Application
------------------------------

### Design# Directory Structure

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

The directories such as docs, view, and var, should be configurable during the boot up process. 

### Design# Union File Manager

To construct $app based on the environment, such as production, local, staging, etc. use the "union file manager" thing. A lot of framework uses it but I cannot find the name of the technique. Nothing special here, as well. 

### Design# No DI Container Component

The Union File Manager controls the construction process, i.e. it is the DI container. No external DI container, such as Pimple, are used. In other words, I will create my own simple container. 

Once the DI process has a standard, or de-fact standard arises, I would like to use it, though. 

### Design# Factory Method: Forge

Use static ```forge``` method as standard factory method for various classes. Ideally, the factory method will not need any arguments. 




