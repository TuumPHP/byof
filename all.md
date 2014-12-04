フレームワークを自作しよう
=====================




HTTPリクエストとリスポンス概要設計
===============

SymfonyのHttp-Foundationのリクエストとレスポンスを元に設計してみる。
まずは簡単な、レスポンスの設計から。

Response概念設計
--------------

ここはLaravel4.2のレスポンスAPIを参考に設計してみる。自分が知っているFWであること、LaravelのAPIは使いやすいので、ぜひ参考にしたい。

大きくレスポンスには、テンプレートを使ってHTMLを返すView、そして別URLに飛ばすRedirectの2種類について考える。

### View.php

最初はテンプレートView用レスポンス。
テンプレートファイル名を設定する。レンダーリングはテンプレートエンジンに任せるので、レスポンスとしては一切関知しない。

```php
class View extends Symfony\Component\HttpFoundation\Response {
    use ResponseWithTrait;
    protected $file;
    public function setFile( $file ) {
        $this->file = $file;
    }
    public function getFile() {
        return $this->file;
    }
}
```

### Redirect.php

次はリダイレクト。

```php
class Redirect extends Symfony\Component\HttpFoundation\RedirectResponse {
    use ResponseWithTrait;
}
```

### ResponseWithTrait

両方共ResponseWithTraitが使われています。
withから始まるメソッドを実装してます。

レスポンスオブジェクトにはデータを登録する必要があります。Viewならテンプレートエンジンに、Redirectならセッションにデータを渡して必要な処理を行ってもらいます。これを行うためのtraitを作ります。

```php
trait ResponseWithTrait {
    protected $data;
    public function with($key, $value ) {
        $this->data[$key] = $value;
    }
    public function withMessage($value ) {
        $this->data['message'] = $value;
    }
    public function withErrorMsg($value ) {
        $this->data['error'] = $value;
    }
    public function withValidationMsg(array $value ) {
        $this->data['errors'] = $value;
    }
    public function withInput(array $value ) {
        $this->data['input'] = $value;
    }
    public function getData() {
        return $this->data;
    }
}
```

こんなかんじかな。


### Responderファクトリ

他にもJsonを返したい、とかありますね。
そして簡単なFactoryオブジェクトが欲しいです。

```php
class Responder {
    public function view( $file ) {
        $response = new View();
        $response->setFile( $file );
        return $response;
    }
    public function redirect( $url ) {
        $response = new Redirect( $url );
        return $response;
    }
    public function json(array $data ) { // 怪しい
        return new Response( json_encode($data), Response::OK, [
            'Content-Type' => 'text/json',
        ] );
    }
}

// 使い方
$respond = new Responder();
$respond->view('template.file.name'); // view
$respond->redirect('http://www.example.com/test'); // redirect
$respond->json($jsonData); // json
```

コードはサンプルですが。

### Requestとの関連性

しかし、現在のpathinfoからの相対パスで指定したいと思いませんか？
すると、こんなメソッドが必要でしょう。

```php
class Responder {
    public function to( $url ) {
        $url  = $this->request->getUriForPath( $url );
        $response = new Redirect( $url );
        return $response;
    }
}
```

このコードで動くかどうかはテストしないといけませんが、_リクエストオブジェクトが必要_ になることが分かります。

つまり、RedirectはRequestに依存することになります。
確かにリクエストに対してレスポンスを返すことを考えると、依存性は理解できます。

Request概念設計
--------------

Request側にリクエストごとに変わる変数を寄せる。
またレスポンスを返すのに必要なサービスを持つこととする。

今のところ、必要なサービスはResponderのみです。ついでにセッションも設定してしまいましょう。

```php
use Symfony\Component\HttpFoundation\Request as SymfonyRequest;
use Symfony\Component\HttpFoundation\Session\Session;
class Request extends SymfonyRequest {
    public static function start( $storage = null ) {
        $request = new Request( $_GET, $_POST, array(), $_COOKIE, $_FILES, $_SERVER );
        $request->setSession( new Session( $storage ) );
        return $request;
    }
}
```


### レスポンス生成

先のレスポンダーを設定できるようにします。簡単に。

```php
class Request extends SymfonyRequest {
    protected $respond;
    public function setResponder($responder) {
        $this->respond = $responder
        $responder->setRequest($this);
    }
    public function respond() {
        return $this->respond;
    }
}
```

どうやってResponderを注入するか…

### setup method

こんな方法で？

```php
class Request {
    public function setup() {
        $this->setRespond(new Responder($this));
        return $this;
    }
}
// このタイミングかな。
$request  = Request::start()->setup();
$response = $app->call($request)
```

他にRequestオブジェクトが必要なサービスは、この方法で作るか…

サービスが散らばるなぁ。



### コンテナが欲しい



これだけで十分でしょうか？

今後の設計で必要なことが増えてくると思います。


アプリケーションの設計
================

Unionファイルマネージャーでの設定
-------------------

アプリケーションを構築する際の要件として、環境により微妙に設定を変える必要があります。要はサーバーが、本番なのか、ステージング、あるいは開発環境なのかによって、エラーやログの取り方が変わってくるわけです。

```php
app/config
app/config/local
```

### ファイル構成

```
root
  +- app/
  |- public/
  |- src/
  |- var/
  +- vendor/
```

*   app: 設定ファイルなど。
*   public: ウェブサイト用。
*   src: サイト用のクラス
*   var: 一時ファイル、ログ、キャッシュなど。コード管理外。
*   vendor: 依存コンポーネント by Composer。コード管理外。

```
app
  +- config/
  |    |- local/
  |    +- stage/
  |- views/
  |- app.php
  +- boot.php
```

*   config: 設定用ディレクトリ。
*   config/{local|stage}: 各環境での設定ファイル。
*   views: テンプレートファイル。


アプリ実行
--------

### アプリケーションの構築：boot.php



```php
$config = new Locator(__DIR__.'/config');
$environment = '';
if(file_exists(__DIR__.'/.env.php')) {
    $environment = include(__DIR__.'/.env.php');
    $config->addRoot(__DIR__.'/config/'.$environment);
}
$app = App::start();
$app->register('environment', $environment);
$app->register('config',   $config);
$app->register('template', $config->php('template'));
$app->register('logger',   $config->php('logger'));
$app->register('respond',  $config->php('responder'));

$app->push($config->php('error_stack'));
$app->push($config->php('session_stack'));
$app->push($config->php('view_stack'));
$app->push($config->php('routes'));
$app->push($config->php(''));
return $app;

```

### 実行：app.php

```php
$app = include(__DIR__.'/boot.php');
$request  = Request::start();
$response = $app->call($request);
$response->render();
```

Service
------

```php
$app->setService( $name, $service );
$service = $app->$servce();
```

Filter
------

こう。

```php
$app->setFilter( $name, $filter );
$response = $app->filter($name, $request);
```

でも、こう使えるといいな。

```php
$response = $request->filter($name);
```

でも、Requestに機能が入りすぎている。
$app（不変）と$request（リクエストごとの変数）という分類にそぐわない。


Controllerの概要
===============

ControllerもHttpスタックとして考える。
ただし、Controllerはリクエストの度に構築する。つまり厳密にはスタックの一部ではないけれど、同じインターフェースを持っている、とする。

あとは、マイクロフレームワークを参考に、クロージャーでもOKとする。


Controllerクラス
---------------

まずは、こう。

```php
class MyController implements RequestStackInterface {}
```

### forge

DIコンテナを使わないという方針を建てた。ので、クラス名からオブジェクトを生成する方法を考えないといけない。

DIコンテナを使うとした場合は、DIコンテナを使えばよいので、こちらもフックはしておくべきだろう。
DIコンテナがない場合は、ファクトリメソッドを使うことを考える。

```php
abstract class AbstractController implements RequestStackInterface {
    /**
     * overwrite this method!
     */
    public static function forge() {
        return new static();
    }
}
```


### Dispatching

```php
function dispatch( $request, $class )
{
    if( $class instanceof \Closure ) {
        return $class( $request );
    }
    if( $dic = $app->dic() ) {
        $controller = $dic->get( $class );
    } else {
        $controller = $class::forge();
    }
    return $controller->call( $request );
}

```

さて、callの中でディスパッチできるようにする。

```php
function call( $request )
{
    $method = $request->getMethod();
    $method = 'on' . ucwords($method)
    return $this->$method( $request );
}
```

という感じにしたい。が、一つ気に食わないことが。




Routing
--------

FastRoute使ってみるかな。
Aura.Routeもテストしたい。


Dispatching
-----------

### DIコンテナ欲しい


### 超簡易版DIフォージャー


Controllerクラスの決定



Viewの概要
=========

Template Engine
---------------

生PHPで。

### EngineInterface

他のテンプレートエンジン使えるように。

### View_Handle




MVCで考えてみる
=======


つまりインフラストラクチャー（ようはデータベース）、あるいはモデル側に関しては今回のフレームワークの検討外とします。

MVCで言えば、VCの部分。

ついでにVとCを分離して考えたい。




必要なパッケージの選定
アプリのインスタンス化の方法
ミドルウェアの設計
ディスパッチャの設計
コントローラーの設計
ビューの設計
各ミドルウェアのデータ共有


