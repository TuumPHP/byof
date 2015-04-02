Build Your Own Framework (BYOF) using PHP
=========================================

TuumPHPの設計覚書。

ウェブアプリケーションとは何か？
--------

まず最初は、ウェブアプリケーションとは何か？から考えます。色々な答えがありますが、Symfonyなどでも使われている一番簡単な答えを元にします。

```php
$response = $app($request);
```

ここで、

*   ```$app```がウェブアプリケーション、
*   ```$request```がHTTPリクエスト、
*   ```$response```を返します。

これだけ！

この３つのについて、考えてゆきます。


##### [X] コンポーネントベース

コンポーネントの組合せでフレームワークを構築します。今や、珍しくもありませんね。

##### [X] Middlewareをベースに

[StackPHP](http://stackphp.com/) で有名になりましたが、ミドルウェアをベースにアプリケーションを構築します。できるだけミドルウェアを使って実装します。

##### [X] 疎結合

コンポーネント間の関係を出来るだけ疎結合にします。えぇと、疎結合にできたらいいな、ということです。

##### [X] データの流れを明確に

リクエストで受け取ったデータが、最後にレスポンスに渡されるまでの流れが明確になるように、データの流れとオブジェクトの関係を一致させる。

##### [X] PSR-7 Http Message

Psr-7が出てきましたので、これを使います。Psr−7では、これらのオブジェクトも不変オブジェクトとなりますが、オブジェクトを「乗り継ぎ」ながら、次々と処理を行うことになります。

##### [X] 不変オブジェクト

バグの減らすため、できるだけ状態を持たないことを目標とします。

まずは、アプリケーション```$app```について、出来るだけ不変なオブジェクトとして設計します。実行の前と後で状態が変わらないのが目標です。

すると、処理結果などは```$request```か```$response```が持つことになります。

> PHPなので、Thread Safeである必要はないけれど、不変にしておいたほうが、バグが減る？設計が分かりやすい？など効果があると良いなと思ったため。


ミドルウェアの設計
-----------

##### [X] 単純なCoRパターンを利用する

ミドルウェアとは、単純な機能を持つミドルウェアを次々と実行することで、複雑な機能を実現すること。実装としてはChain of Responsibilityパターンを用います。

### アプリケーションAPI

基本的なCoRを用いるため、入力```$request```に対して、出力```$response```を返すこととします。またレスポンスが帰ってきた時点でチェーンを終了します。

```php
$response = $app($request);
```

振る舞いとしては、```$request```を受け取り、対応した場合は```$response```を返す。あるいは次のミドルウェアを実行する。なお、次のミドルウェアが無ければnullを返す。

このAPIを使った理由としては、

*   APIとして（おそらく）もっとも簡潔。
*   返り値にたいしての処理を簡単に行える。
*   往路・復路とも、必ず通過するミドルウェアを設定可能。


> このAPIを採用した理由ですが、他には…
>
> *   Symfonyで慣れている。
> *   Strutsのような```$app($request, $response);```のAPIも考えたが、Psr-7に```Response::withAttribute```が無く、データを持ち回すのに使いづらかった。
> *   Conduitのような```$app($request, $response, $next);```は単純に```$next```が面倒そうで…
>
> と決定的な理由は考えられなかったです。単に慣れてただけという話も。また、メソッドを使ってもいいのですが、メソッド名を考えなくていいクロージャーとして設計を進めます。



### ミドルウェアAPI

ミドルウェアが自身でレスポンスを返さない場合は、次のミドルウェアを呼び出す必要があります。その次のミドルウェアを設定するAPIとしては、メソッドを用いた簡単な方法を使います。

```php
// チェーンの最後にミドルウェアを追加する。
$middleware->push($nextMiddleware);
// チェーンの最初にミドルウェアを追加する。
$middleware->prepend($nextMiddleware);
```

以上、3つのメソッドを持つインターフェースとなります。

#### 別アプリケーションの場合

ミドルウェアだけれど、次のミドルウェアを呼びだす機能を持たないオブジェクトを利用することを可能にします。例えば、CSRFトークンチェックや認証ルーチンなどの場合、次のように簡単に利用できるようにします。

```php
$middleware = new Middleware(new AdminAuthFilter());
$middleware->when('/admin*');
$app->push($middleware);
```

メリットとしては、```AdminAuthFilter```を実行する条件を持たないので、オブジェクトの使い回しが簡単に出来ると考えます。

> 実行する条件を細かく設定したい場合、ミドルウェア自身に条件を持つより、別ミドルウェアから呼び出されたほうが簡潔になると考えたため。また、$appが不変と考えると、条件によって適用するミドルウェアはチェーンから外すほうが良いかなと考えたため。
 
ただし、別アプリケーション内でリクエストを修正したい場合に問題となります。これは$requestが不変オブジェクトのためで、例えば新たにアトリビュート追加した場合、$requestをチェーンに返す必要があります。

このため、APIに```$next```という変数を追加します。この変数の利用方法としては、レスポンスは返さないが、次の処理で使われる（かもしれない）情報を```$request```に追加するために用います。

```php
class AdminAuthFilter implements FilterInterface {
    public function __invoke($request, $next=null) {
        if($auth = $this->authOK($request)) {
            $request = $request->withAttribute('auth', $auth);
            return $next ? $next($request) : null;
        }
    }
}
```

### ApplicationInterface and MiddlewareInterface

以上から、次の2つのインターフェースを定義しました。

```php
interface ApplicationInterface
{
    /**
     * @param Request          $request
     * @param callable|null    $next
     * @return null|Response
     */
    public function __invoke($request, $next=null);
}

interface MiddlewareInterface extends ApplicationInterface
{
    /**
     * stack up the SplStack.
     * converts normal HttpKernel into Stackable.
     *
     * @param ApplicationInterface $handler
     * @return $this
     */
    public function push($handler);

    /**
     * prepends a new middleware/application at the
     * beginning of the stack. returns the prepended stack.
     *
     * @param ApplicationInterface $handler
     * @return MiddlewareInterface
     */
    public function prepend($handler);
}
```


リクエストとレスポンスの設計
----------------------------

### 基本方針

##### [X] Psr-7の活用

今のところPsr-7は確定してませんが、ほぼ最終形ということなので、Psr-7を元にリクエストとレスポンスを設計します。特徴は、両方共普遍オブジェクトとして設計されている点にあります。

一方、アプリケーション$appも普遍として構築するので、リクエストごとに異なる値や処理中に変更・追加される値はリクエストまたはレスポンスで保持する必要があります。

### Request設計

リクエストごとに異なる情報・オブジェクトはリクエストに保持することにします。

毎回異なる情報としては、

*   レスポンス、
*   セッション、
*   ログ（現状、対応していない）、
*   ビュー

などが考えられます。

全てをリクエストに保持することは無いですが、アクセス毎に異なる情報を扱う場合は```$app```が変化しないコーディングを行います。

##### [X] リクエストからレスポンスを生成

レスポンスを作成する際に、リクエストの中のアトリビュートを使って構築する場合が多々あると考えられます。例えば、認証情報をレスポンスで使う、現状のURLにリダイレクトする、などです。

したがって、レスポンスはリクエストから生成することにします。

```php
$response = $request->respond(['auth'])->asView('view/file');
```

ここで、respondはレスポンスファクトリを返します。respondで、リクエストのアトリビュートからレスポンスに渡したいkeyを指定することが出来ます。

> 簡単なフィルターなどでも、レスポンスを帰す場合があることを考えると、簡単にレスポンスを生成できるのは便利かと思ってます。

#### その他のサービス

リクエストが管理すべきと思われるサービスを列挙。普通はアプリ側で持つと思うが、考えてみればアクセス毎に異なる状態を持つとかんがえられる。

*   ロガー：アクセス毎に異なるログを出力するため。
*   セッション：アクセス毎に異なる。


### Response設計

レスポンスには大きく3つの種類を検討します。

*   正常なレスポンス。例えばHTMLやJSONのボディを伴う。（ステータスが200、コンテントタイプが異なる）。
*   リダイレクト。（ステータスが300番台）。
*   エラー。（ステータスが400または500番台）。

##### [X] 簡単なレスポンスファクトリを用意する

正常レスポンス。ファイルのダウンロードなどの便利メソッドが欲しいかな。

*   asHtml(string $html):
*   asView(string $view_file):
*   asText(string $text):
*   asJson(array $data):

リダイレクト系。もう少し種類は欲しい。

*   toAbsoluteUri(UriInterface $uri):
*   toPath(string $path):
*   toBasePath(array $parameter=[]);
*   toNamedRoute(string $name, array $parameter=[]); （未対応）

エラー系。これも種類が足りない。

*   asError(int $status):
*   asNotFound():
*   asForbidden():

##### [X] 同じAPIでデータを設定できる。

これはLaravelのマネです。レスポンスの種類が異なっても、同じAPIでデータを設定します。

```php
$response = $request->respond()
    ->with($key, $value)
    ->withInputData($input_data)
    ->withInputErrors($validation_errors)
    ->withMessage($message)
    ->withNotice($notice_message)
    ->withError($error_message)
    ->as{Type}();
```

> データの設定に、```with```から始まるメソッドを使ってますが、respond()でかえってっくるレスポンスファクトリは不変オブジェクトでは無いです。Laravelを参考にした名前です。

レスポンスの種類により、データの使われ方が異なります。

#### レスポンスとミドルウェア

CoRの復路において、ミドルウェアがレスポンスの種類に応じた処理を行います。

*   __正常レスポンス：__ Viewの場合はViewStackでテンプレートのレンダリングを行う。
*   __リダイレクト系：__ リダイレクトに設定されたデータはセッションのフラッシュに登録しておいて、次のアクセス時に取り出すことにする。
*   __エラー系：__ エラー用ミドルウェアが、エラー種類に応じたテンプレートをレンダリングする。
*   __その他：__ テキストならテキストとして、配列ならJSONとしてレスポンスを自動で生成するようにする。

> レンダリングをレスポンスあるいはコントローラーで行うことを考えたのですが、エラー処理やフィルターからのレスポンスを考えるとミドルウェアで行うのが一番広くカバー出来ると考えたため。

ウェブアプリケーションの基本構成
---------------------------

標準として用意する基本ミドルウェアについて。

### 基本ミドルウェア

#### error-stack

*   往路：whoopsも構築＆設定する。が、これは本来はミドルウェア内で行う処理ではない（ので、移動する予定）。
*   復路：レスポンスがErrorタイプの場合、エラーレスポンスを描画する。レスポンスが無い（null）の場合はnotFoundを描画する。描画はErrorViewオブジェクトを使うことで、エラーステータスに応じたテンプレートを描画する。

#### session-stack

*   往路：セッションの設定（configureで行うべきかも。）フラッシュの内容をリクエストのアトリビュートに複製する。
*   復路：レスポンスがRedirectタイプの場合、レスポンスのデータをセッションのフラッシュに保存する。

#### csRf-stack

*   往路：CsRfトークンを設定する。ルートがマッチする場合は、CsRfトークンを確認する。（トークンの設定処理はconfigureで行うべきかも。）
*   復路：特になし。

#### view-stack

*   往路：特になし。
*   復路：レスポンスがViewタイプの場合、描画する。レスポンスがstringならテキストとして、配列ならJSONとして描画する。

#### url-mapper-handler

*   往路：URLから直接マップするファイルを特定して、存在したら描画する。必要なければスタックに入れないこと。
*   復路：特に無し。

##### [_] SessionとCsRfスタックを統一する？


ディレクトリ構造
------------------

### ディレクトリ構成

現状のディレクトリ構成。

```sh
project-root
  |- app/          # アプリケーション設定スクリプト
  |- public/       # document root
  |- src/          # PHPクラス
  |- var/          # ログ、キャッシュなど
  +- vendor/       # composer！
```

さらに、```app/```以下のディレクトリ構成。

```sh
app
|- config/         # 設定用DI、スクリプト
|- documents/      # 静的ファイル
|- migrations/     # Phinx
|- utils/          # 色々スクリプト
|- views/          # テンプレート
|- app.php         # アプリケーションの構築と実行
|- config.php      # 設定値
+- routes.php      # ルート設定
```

### DIコンテナ

##### [_] コントローラー生成でDIコンテナを利用すること

実は未だ使っていない。


RouterとController
---------------------

### 独自Router (´・ω・`)

調べているうちに、自分で書いてしまったRouterを利用する。



### コントローラー設計

コントローラーもミドルウェアの一部として実行する。ただし次のミドルウェアを持たないアプリケーションとして扱われるため、```ApplicationInterface```を実装する。ただし、純粋なミドルウェアとしてしまうと、使い勝手が悪いのでtraitを使って便利な機能を提供する。

> なお、アプリケーションのチェーンの一部とはしない（不変にするため）。

##### [_] traitだけで対応できるようにする。

現状は、```AbstractController```を継承する必要がある。

#### 最も基本的なController

```php
class BasicController implements MiddlewareInterface {
    public function __invoke($request) {
    }
}
```

#### Dispatchコントローラー

よく使われそうなオブジェクトをオブジェクトのプロパティとして設定している。

```php
class BasicController extends AbstractController {
    public function dispatch() {
        $input = $this->request->getQuery();
        return $this->respond->toBasePath('/'.$input['id']);
    }
}
```

とはいえ、これでも使いづらいので…

#### メソッドルーティング

httpメソッドを元にしてコントローラー内のメソッドを呼び出す。

```php
class RoutingController implements MiddlewareInterface {
    use DispatchByMethodTrait;
    public function onGet($id) {}
    public function onPost() {}
}
```

*   メソッド名をonから始めるメソッドを呼び出す。
*   Query、つまりGetパラメターをメソッドの引数で受けることが出来る。


#### 内部ルーティング

コントローラー内に、簡易ルーティングを設定する。```getRoutes```メソッドで、ルート一覧を返すと、それに基づいたメソッドを呼ぶ。なおオブジェクトとしてのメソッド名は```on{Method}```となる。

```php
class RoutingController implements MiddlewareInterface {
    use RouteDispatchTrait;
    protected function getRoutes() {
        return [
            'post:/{id:i}' => 'put',
            'get:/{id:i}'  => 'get',
        ];
    }
    public function onGet($id) {
        return $this->respond->asJason(['id'=>$id]);
    }
}
```

Route情報は、URIの絶対パスを使って指定する方法と、Routesでマッチする部分を指定する方法がある。

URLルートでマッチングする場合では、ルートとコントローラーを次のようにマッチさせる。

```php
$routes->any('/Routing*', RoutingController::class);
// use routes like: 'get:/Routing/{id:i}'
```

ベースのURLを使う場合は、最後に```{*}```として、この部分にマッチさせるよう設定する。

```php
$routes->any('/Routing{*}', RoutingController::class);
// use routes like: 'get:/{id:i}'
```


#### リソース型コントローラー

擬似リソース型コントローラーを作成する方法。簡単にいえばRouteDispatchTraitにデフォルトのルートを設定しているだけ。

```php
class RoutingController implements MiddlewareInterface {
    use ResourceControllerTrait;
    public function onGet() {}
    public function onPost() {}
}
```

単にデフォルトの```getRoutesメソッド```を持った```RouteDispatchTrait```です。対応は、[ソースを参照](https://github.com/TuumPHP/Web/blob/master/src/Controller/ResourceControllerTrait.php)して下さい。


ViewとTemplate
------------------

### View用ミドルウェア

レスポンス内にはテンプレート描画用のデータを持つだけとしておいて、View用ミドルウェアがビューの描画を実行する。

> Filterなどからエラーレスポンスを返す場合などを考えた場合、コントローラーやレスポンスなどから独立させてレンダリング出来る方が便利と考えたため。

##### [X] Viewも不変

レンダーするさいのViewオブジェクトも不変である（ありたい）。

##### [X] PHPをテンプレートとして用いる

簡単に開発するならPHPを利用するのが早いと思うので。

でもtwigとかも使ってみたい。

### 標準ヘルパー

respondに対応した標準のヘルパーを用意する。

*   withMessage, withAlert, withError: ```$message```
*   withInput: ```$inputs```
*   withInputErrors: ```$errors```

