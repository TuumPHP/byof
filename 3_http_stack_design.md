HTTPスタックの基本設計
==================

Slim FrameworkあるいはStackPHPで使われている設計には幾つか不満があります。そこで、今回の開発では、新しくHTTPスタックを設計し直します。

StackPHPとHttpKernelInterfaceの問題点
-------

### HTTPスタックとミドルウェアが一体になっている

StackPHPのデザインだと、

*   リクエストを受け取って処理をする、
*   次のスタックを呼び出す、

という責務がひとつのオブジェクトなってしまいます。具体的なコードで書くと、

```php
class Stack {
  protected $app;
  public function __construct($app) {
    $this->app = $pp;
  }
function handle(Request $request, $type, $catch) {
    /* 何か処理を行う */
    // 何も無ければ、次の$appを呼び出す。
    $response = $this->app->handle($request, $type, $catch);
    /* レスポンスにも処理を行える */
    return $response;
  }
}
```
この中で```handle```メソッド内で、実際の処理を行います。レスポンスを返す場合は返しますが、返さない場合は次の処理を実行する必要があります。

確かに、これなら「何でもできる」コードですが、責務が多すぎて、かえって利便性が落ちてしまうのではないかと考えました。

### SymfonyのHttpKernelInterface

このインターフェースは、Requestクラスが指定されてます。
本来なら、Interface（何でもOK）を指定すべきだったのでは、と想像します。


スタックの設計
-------------

ということで、自分で設計です。
とはいえ、大きく変更するわけではありません。

要するに、

*   $requestを受け取って、
*   $responseを返す。返せない場合はnullを返す。

です。これをinterfaceとして実装します。

### StackHandleInterface

```php
interface StackHandleInterface {
  /**
   * @param Request $request
   * @return Response|null
   */
  public function handle($request);
}
```

本当は、```$request```にはinterfaceを指定したいところではあります。
が、今はPSRで共通のHTTPインターフェースの実装が進んでいます。PSRが決定したところで実装できるようにアノテーションでの指定にとどめておきたいと思います。

> handleというメソッド名はSymfonyのHttpKernelInterfaceのマネ。何か格好良いメソッド名ないかな。
> 
> 例えば```__invoke```を使ってクロジャーにしようかとも思いましたが現在のPHPだと今ひとつ上手にパースしてくれないので却下。PHP7に期待したい。

### StackReleaseInterface

さて、もうひとつ。
帰ってきたリスポンスに対する処理を実装したいですね。

```php
interface StackReleaseInterface {
  /**
   * @param Request  $request
   * @param Response $response
   * @return Response|null
   */
  public function release($request, $response);
}
```

こちらは、戻ってきたリスポンスに対する処理です。

> メソッド名のreleaseは工夫したいな。
> callとreplyで、意味がわかるかな。


### Stackable class

これだとスタックしません。ので、各スタックがつながるようミドルウェアに変換するクラスです。

```php
class Stackable implements StackableInterface {
  protected $stack;
  protected $next;
  
  /**
   * 処理を行う$stackを渡して下さい
   */
  public function __construct($stack) {
    $this->stack = $stack;
  }
  
  /**
   * $requestを受け取って、自分の$stackを実行してみます。
   */
  public function handle($request) {
    // $responseがなければ、次の$nextを処理します。
    if( !$response = $this->stack->call($request) ) {
      if( $this->next ) {
        $response = $this->next->call($request);
      }
    }
    // 最後にreleaseメソッドを実行します（あれば）。
    if( $this->next instanceof StackReleaseInterface ) {
      $response = $this->app->release($request, $response);
    }
    return $response;
  }
  
  /**
   * 処理用のスタックをStackableに変換。
   */
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

### ミドルウェアの機能

実際に開発を始めると、色々なアイディアが出てきました。こんな機能があると便利でしょうか。

* matcher

簡単にリクエストのパスとの比較条件をつけることです。
例えばパスが```/admin```で始まってたら、```auth```スタックを実行する、などです。

* before/after filter

マッチできるとなると、マッチしたらフィルターを実行する機能がが欲しくなります。

どうやって実装するかは、また後で検討してみます。


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

になるから今回は却下してみる。


### それなら、コントローラーとは？

コントローラーもHTTPスタックとして考えたい。
ただし、スタックとしてはクラス名で指定しておいてので、必要なときに実体化してから呼び出す。


### コントローラーのメソッドは？

例えばLaravelなら```controller\class\name@methodName```と指定すると、メソッドを走らせてくれる。これはスタックなのか？

