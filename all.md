フレームワークを自作しよう
=====================



フレームワークの基本設計
====================

今回のフレームワークでは、httpリクエストを受け取ってからコントローラを呼び出すところ、それとビュー（Htmlなど）を返す部分について考えます。

基本になるのはStackPHPの考え方です。
これをもとにアプリケーションの概要を考えます。


アプリケーションの概念設計
----------------

### HTTPの基本

想定している、ウェブアプリケーションの使い方です。
SymfonyのHttp-Foundationを基本に考えます。

```php
/*
 * まずはアプリケーションを構築する。
 */
$app = function_to_build_framework();

/*
 * アプリケーションを走らせる。
 */
$request  = Request::create($_SERVER); // 入力はHTTPリクエスト。
$response = $app->run($request); // 実行！
/*
 * $responseが実行結果
 */
$response->render(); // httpリスポンスを返す
```

ここで、
*   $appがアプリケーション、
*   $requestはHTTPリクエスト、
*   $responseがHTTPリスポンス、

です。

$appを最初に構築。構築の仕方は、これから考えるので、ひとまず「function_to_build_framework」という関数で作ることとしてあります。

$appの入力が、HTTPリクエスト、要はSymfonyのRequestオブジェクトですね。HTTPリクエストの情報以外に、PHP独特の設定やセッションなども含むと考えます。

#### デザイン要件１：アプリケーションとリクエストの分離

今回の設計において、$requestと$responseがリクエストごとに異なる値を持ちますが、$appは常に同じ状態を保つことを目指したいと思います。

できれば、処理実行前と後で$appの状態が同じであるのが理想です。

これにより、アプリケーションオブジェクト（$app）とリクエスト＆リスポンスがキレイに分離されることになるのではないでしょうか。

### ミドルウェア

アプリケーションの構成も、StackPHPの考え方を元にします。HTTPスタックをミドルウェアとして次々と実行する形です。

例えば、次のようなHTTPスタックのチェーンを作成します。

```php
class HttpStack {
  protected $next;
  function call($request) {
    // do some job.
    $response = $this->next->call($request);
    // do more job.
    return $response;
  }
}
```

ここでは、$nextが次のミドルウェアで、次々と実行することができます。どこかのミドルウェアでリスポンスが帰ってきたら、チェーンは終わりです。今度は次々とリスポンスを返してゆきます。

> これはChain of Responsibility パターンになると思います。
> 一方、各HTTPスタックで$requestの内容を変更してゆくこともおおいので、decoratorパターンでもあります。



#### デザイン要件２：できるだけHTTPスタックで機能を実現する

ということで、本フレームワークのデザイン要件２は、できるだけHTTPスタックでアプリケーションを構築する、ということです。

全てをスタックで表現できないと思いますが、少なくともURLのルーティングぐらいは十分可能です。

一方でスタックに合わない要素は出てくるはず。その場合は、
*   リクエストごとに異なる場合は$requestまたは$responseで、
*   それ以外は$appで、

実装します。

この「それ以外」とは、何があるのでしょうね。

思いつくのは、
*   設定、
*   イベンt・フック、
*   サービス、フィルター、もろもろのコンテナ、
*   テンプレート・ビュー、

あたりでしょうか。このへんも、後で設計したいと思います。

### その他、必要な機能

コンフィグ
フィルター
サービス
DIコンテナ
テンプレートエンジン


HTTPスタックとミドルウェアの概念設計
--------------------------

StackPHPの設計もいいのですが、幾つか不満があります。
今回の開発では、新しくミドルウェアを設計し直します。


### HTTPスタック設計

HTTPスタックの要件は、
*   $requestを受け取って、
*   $responseを返す。返せない場合はnullを返す。

です。これをinterfaceとして実装します。

> HTTPスタックはミドルウェアではありません。次のアプリケーションを呼び出す責務は、ミドルウェアにあり、両者を明確に区別することにします。

では、インターフェースです。

```php
interface RequestStackInterface {
  /**
   * @param Request $request
   * @return Response|null
   */
  public function call($request);
}
```

SymfonyのHttpKernelInterfaceのように、$requestにクラスあるいはinterfaceを指定したいところではあります。が、今はPSRで共通のHTTPインターフェースの実装が進んでいます。PSRが決定したところで、実装できるようにアノテーションでの指定にとどめておきたいと思います。

> 何か格好良いメソッド名ないかな。__invokeを使ってクロジャーにしようかとも思いましたが現在のPHPだとパースが今ひとつなので却下。

さて、もうひとつ。
リスポンスに対する処理を実装したいですね。

```php
interface ResponseStackInterface extends RequestStackInterface {
  /**
   * @param Request  $request
   * @param Response $response
   * @return Response|null
   */
  public function reply($request, $response);
}
```

こちらは、リスポンスに対する処理です。

> メソッド名は工夫したいな。
> callとreplyで、意味がわかるかな。



### ミドルウェア概要設計

HTTPスタックをミドルウェアに変換するクラスです。

```php
class MiddlewareInterface implements RequestStackInterface {
  protected $stack;
  protected $next;
  public function call($request) {
    $response = null;
    if( !$response = $this->stack-> call($request) ) {
      if( $this->next ) {
        $response = $this->next-> call($request);
      }
    }
    if( $this->app instanceof ResponseStackInterface ) {
      $response = $this->app->reply($request, $response);
    }
    return $response;
  }
}

```

同じくRequestStackInterfaceを実装することで、パッと見はHTTPスタックと同じですが、実際の処理は、別途のメンバー$stackが実行します。

結果が帰ってこなければ、次のスタックを実行します。

またResponseStackInterfaceの場合は、さらにリスポンスに対して処理を行います。


### 全部合わせて

さすがにミドルウェアは簡単すぎます。
ので、こちらもインターフェースを作成しましょう。

まずは、次のミドルウェアを受け取ります。pushでいいかな。

```php
interface MiddlewareInterface extends RequestStackInterface {
  public function push($stack);
}
```

ミドルウェア自身に簡単なルーティングを実装するのはどうでしょう？行き過ぎ？

```php
interface MiddlewareInterface extends RequestStackInterface {
  public function onRoute($url);
  public function onMethod($method);
}
```

そして、beforeとafterフィルター。
上記のルーティングにマッチした場合のみ実行。

```php
interface MiddlewareInterface extends RequestStackInterface {
  public function before($stack);
  public function after($stack);
}
```

ちょっと考えなおす。


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


