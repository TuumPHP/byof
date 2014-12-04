HTTPスタックの基本設計
==================

StackPHPには幾つか不満があります。

今回の開発では、新しくHTTPスタックを設計し直します。

StackPHPとHttpKernelInterfaceの問題点
-------

### HTTPスタックとミドルウェアが一体になっている

StackPHPのデザインだと、
*   リクエストを受け取って処理をする、
*   次のスタックを呼び出す、

という責務がひとつのオブジェクトなってしまいます。

これだと使いにくいのと利便性が落ちてしまうのではないかと考えました。

### SymfonyのHttpKernelInterface

このインターフェースは、Requestクラスが指定されてます。
本来なら、Interface（何でもOK）を指定すべきだったのでは、と想像します。


TuumStackの設計
-------------

ということで、自分で設計です。
とはいえ、大きく変更するわけではありません。

要するに、
*   $requestを受け取って、
*   $responseを返す。返せない場合はnullを返す。

です。これをinterfaceとして実装します。

```php
interface StackCallInterface {
  /**
   * @param Request $request
   * @return Response|null
   */
  public function call($request);
}
```

本当は、$requestとしてinterfaceを指定したいところではあります。
が、今はPSRで共通のHTTPインターフェースの実装が進んでいます。PSRが決定したところで実装できるようにアノテーションでの指定にとどめておきたいと思います。

> callはRackのマネ。何か格好良いメソッド名ないかな。
> 例えば```__invoke```を使ってクロジャーにしようかとも思いましたが現在のPHPだとパースが今ひとつなので却下。

さて、もうひとつ。
リスポンスに対する処理を実装したいですね。

```php
interface StackReplyInterface extends RequestStackInterface {
  /**
   * @param Request  $request
   * @param Response $response
   * @return Response|null
   */
  public function reply($request, $response);
}
```

こちらは、戻ってきたリスポンスに対する処理です。

> メソッド名は工夫したいな。
> callとreplyで、意味がわかるかな。


### ミドルウェア概要設計

HTTPスタックをミドルウェアに変換するクラスです。

```php
class Middleware implements StackCallInterface {
  protected $stack;
  protected $next;
  
  public function __construct($stack) {
    $this->stack = $stack;
  }
  
  public function call($request) {
    if( !$response = $this->stack->call($request) ) {
      if( $this->next ) {
        $response = $this->next->call($request);
      }
    }
    if( $this->next instanceof StackCallInterface ) {
      $response = $this->app->reply($request, $response);
    }
    return $response;
  }
  
  public function push($next) {
    if( $this->next ) {
      return $this->next->push($next);
    }
    if( !$next instanceof Middleware ) {
      $next = new self($next);
    }
    return $this->next = $next;
  }
}
```

同じくRequestStackInterfaceを実装することで、パッと見はHTTPスタックと同じですが、実際の処理は、別途のメンバー$stackが実行します。

結果が帰ってこなければ、次のスタックを実行します。

またResponseStackInterfaceの場合は、さらにリスポンスに対して処理を行います。



様々なミドルウェア
--------------

後で考える。

ミドルウェアに色々な機能をつけたくなりました。
こういう機能があると便利でしょうか。

### matcher

簡単にリクエストのパスとの比較条件をつけることです。
例えばパスが```/admin```で始まってたら、```auth```スタックを実行する、などです。

実行するスタックは常に同じ。

```php
class Middleware {
  public function onRoute($url) {}
  public function onMethod($method) {}
}
```


### router

次はルーター機能です。
先のmatcherより複雑な条件を指定可能で、実行するスタックを別々に指定します。

```php
class Middleware {
  public function on($method, $url, $stack) {}
  public function onGet($url, $stack) {}
  public function onPost($url, $stack) {}
  public function onDelete($url, $stack) {}
}
```


### before/after filter

ルーター機能を導入すると欲しいのがbefore/afterのフィルターです。

```php
class Middleware {
  public function before($stack);
  public function after($stack);
}
```

スタックとは何か
=============

さて、ここで、もう一度、スタックとは何か？を考えてみる。

スタックとはHttpStackInterfaceを実装したオブジェクトのこと。


### フィルターとは？

フィルターとは、実はHTTPスタックのこと。

ただしフィルター登録されていて、フィルター名で実行が可能になっている。
フィルターの設計はこれから。


### では、クロージャーは？

マイクロフレームワークを見ると、クロージャーも使いたい。
なので、クロージャーもスタックとして可能ということで。


というか、本来は、これが正しい実装な気がする。

ただ、PHPだと

```php
function runClosure() {
  $this->closure = function(){ return 'done';};
  return $this->closure(); // error!
}
```

になるからなぁ。


### それなら、コントローラーとは？

コントローラーもHTTPスタックとして考えたい。
ただし、スタックとしてはクラス名で指定されているので、これを実体化してから呼び出す。


### コントローラーのメソッドは？

Laravelなら```controller\class\name@methodName```としてスタック？を指定すると、メソッドを走らせてくれる。こういう機能。

う〜ん。
欲しいけど、実装が面倒になりそう。
ルーターのみ対応は可能ではあるけれど。

