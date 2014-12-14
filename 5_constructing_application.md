アプリケーションの構築
================

アプリケーションの詳細。

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
  |- .env.php
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
$app = App::start($environment);
$app->register('config',   $config);
$app->register('viewer',   $config->php('viewer'));
$app->register('logger',   $config->php('logger'));
$app->register('respond',  $config->php('respond'));

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
$request  = $app->config()->php('request');
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


