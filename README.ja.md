Build Your Own Framework (BYOF) using PHP
=========================================

TuumPHPの設計覚書。

ウェブアプリケーションとは何か？
--------

まず最初は、ウェブアプリケーションとは何か？からです。色々な答えがありますが、Symfonyなどでも使われている一番簡単な答えを元にします。

```php
$response = $app($request);
```

ここで、

*   ```$app```がウェブアプリケーション、
*   ```$request```がHTTPリクエスト、
*   ```$response```を返します。

これだけ！
この３つのについて、考えてゆきます。


#### [X] コンポーネントベース

コンポーネントの組合せでフレームワークを構築します。

#### [X] 疎結合

コンポーネント間の関係を出来るだけ疎結合にします。

### [X] Middlewareをベースに

[StackPHP](http://stackphp.com/) で有名になりましたが、ミドルウェアをベースにアプリケーションを構築します。できるだけミドルウェアを使って実装します。

### [X] PSR-7 Http Message

Psr-7が出てきましたので、これを使います。Psr−7では、これらのオブジェクトも不変オブジェクトとなりますが、オブジェクトを「乗り継ぎ」ながら、次々と処理を行うことになります。

### [X] 不変オブジェクト

バグの減らすため、できるだけ状態を持たないことを目標とします。

まずは、アプリケーション```$app```について、出来るだけ不変なオブジェクトとして設計します。実行の前と後で状態が変わらないのが目標です。

すると、処理結果などは```$request```か```$response```が持つことになります。


ミドルウェアの設計
-----------

ミドルウェアとは、単純な機能を持つミドルウェアを次々と実行することで、複雑な機能を実現すること。実装としてはChain of Responsibilityパターンを用いる。

### ミドルウェアAPI

基本的なCoRを用いるため、入力```$request```に対して、出力```$response```が決定した時点でチェーンを終了します。

```php
$response = $middleware($request);
```

インターフェースとしては

```php
interface MiddlewareInterface {
    /**
     * @param Request $request
     * @return null|Response
     */
    public function __invoke($request);
}
```

振る舞いとしては、```$request```を受け取り、対応した場合は```$response```を返す。あるいは次のミドルウェアを実行する。なお、次のミドルウェアが無ければnullを返す。この```Request```と```Response```についてはPsrｰ7をベースにした独自クラスとなる

このAPIを使った理由としては、

*   APIとして（おそらく）もっとも簡潔。
*   返り値にたいしての処理を簡単に行える。
*   往路・復路とも、必ず通過するミドルウェアを設定可能。


> このAPIを採用した理由ですが、他には…
>
> *   Symfonyで慣れている。
> *   Strutsのような```$app($request, $response);```のAPIも考えたが、Psr-7に```Response::withAttribute```が無く、データを持ち回すのに使いづらかった。
> *   Conduitのような```$app($request, $response, $next);```は単純に$nextが面倒そうで…
>
> と決定的な理由は考えられなかったです。単に慣れてただけという話も。また、メソッドを使ってもいいのですが、メソッド名を考えなくていいクロージャーとして設計を進めます。


### チェーン追加用API

ミドルウェアが自身でレスポンスを返さない場合は、次のミドルウェアを呼び出す必要があります。その次のミドルウェアを設定するAPIとしては、メソッドを用いた簡単な方法を使う。

```php
// チェーンの最後にミドルウェアを追加する。
$middleware->push($nextMiddleware);
// チェーンの最初にミドルウェアを追加する。
$middleware->prepend($nextMiddleware);
```

### フィルター

次のミドルウェアを呼びだす機能を持たない、ミドルウェアのようなオブジェクトを利用することを可能にしたい。具体的には、CSRFトークンチェックや認証ルーチンを考えていて、例えば次のように利用する。

```php
$middleware = new Middleware(new AdminAuthFilter());
$middleware->when('/admin*');
```

> 実行する条件を細かく設定したい場合、ミドルウェア自身に条件を持つより、別ミドルウェアから呼び出されたほうが簡潔になると考えたため。また、$appが不変と考えると、条件によって適用するミドルウェアはチェーンから外すほうが良いかなと考えたため。
 
ただし、FilterのAPIは、ミドルウェアのAPIとは少し異なる。

```php
interface FilterInterface {
    /**
     * @param Request $request
     * @param null|callable $next
     * @return null|Response
     */
    public function __invoke($request, $next=null);
}
```

パラメータの```$next```の利用方法としては、レスポンスは返さないが、次の処理で使われる（かもしれない）情報を```$request```に追加するために用いる。

```php
class AdminAuthFilter implements FilterInterface 
{
    public function __invoke($request, $next) 
    {
        if($auth = $this->authOK($request)) {
            $request = $request->withAttribute('auth', $auth);
            return $next ? $next($request) : null;
        }
    }
}
```

これは$requestが不変オブジェクトのため、に新たにアトリビュートを入れた$requestをチェーンに返すために用いる。



XXXXXXX
========

### [X] Middlewareと実処理の分離

実際に処理を行うオブジェクト（```$app```）と、ミドルウェアとしてチェーンの構築・実行を行うオブジェクト(```$middleware```)を分離します。

```php
$middleware = new Middleware($app);
```


### リクエスト

Psr-7を設計している人のリクエストを元に作成します。

さて、セッションのようにリクエストごとに異なるデータや、処理していて変更・追加される情報は全部```$request```が持つことになります。


### リクエストへのリスポンス

使いやすいリクエストについて考えます。

例えば、リダイレクトを行う場合、前のURLに戻る、あるいはパスだけ（ホスト名などを省略する）などがありえます。多少ですが、レスポンスはリクエストに依存すると考えます。

情報の流れとオブジェクトの関係を対応させたいので、下記のような方法でレスポンスを生成することにします。

```php
$request->responseFactory()->responseType();
```


### Loggerもリクエストに





