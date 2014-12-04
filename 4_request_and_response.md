HTTPリクエストとリスポンス概要設計
===============

SymfonyのHttp-Foundationのリクエストとレスポンスを元に設計してみる。

もうひとつ、Laravel4.2を参考に設計してみる。自分が知っている（唯一の）フレームワークであること、そして何と言ってもLaravelのAPIは使いやすいので、ぜひ参考にしたい。

Response概念設計
--------------

まずは簡単な、レスポンスの設計から。

大きくレスポンスには、テンプレートを使ってHTMLを返すView、そして別URLに飛ばすRedirectの2種類について考える。

両者とも、withで始まるメソッドで値を設定できることにする。

### View.php

最初はテンプレートView用レスポンス。

```php
class View extends Symfony\Component\HttpFoundation\Response {
    public function setFile( $file ) {}
    public function with( $key, $value ) {}
    public function withInput( array $input ) {}
    public function withMessage( $message ) {}
    public function withErrorMsg( $error ) {}
    public function withInvalidMsg( array $errors ) {}
    public function getFile() {}
}
```

テンプレートファイル名を設定する。レンダーリングはテンプレートエンジンに任せるので、レスポンスとしては一切関知しない。

setFileでテンプレートを指定、後はwithと関連メソッドで必要な値を設定する。

HTTPスタックのどこかで、Viewレスポンスが帰ってきたらテンプレートエンジンに受け渡す、という処理を行います。


### Redirect.php

次はリダイレクト。

```php
class Redirect extends Symfony\Component\HttpFoundation\RedirectResponse {
    public function with( $key, $value ) {}
    public function withInput( array $input ) {}
    public function withMessage( $message ) {}
    public function withErrorMsg( $error ) {}
    public function withInvalidMsg( array $errors ) {}
}
```

やはり、HTTPスタックのどこかで、Redirectレスポンスが帰ってきたら、データをセッション・フラッシュに設定する、という処理を行います。


### ResponseWithTrait

ViewもRedirectもwithに関しては同じ処理になりそうなので、Traitで実装してみたい。

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

これだけあれば十分かな。

### Respondファクトリ

他にもJsonを返したい、とかありますね。

そして簡単なFactoryオブジェクトが欲しいです。
そこで、Respondというファクトリクラスを作ってみます。

```php
class Respond {
    public function view( $file ) {
        $response = new View();
        return $response->setFile( $file );
    }
    public function redirect( $url ) {
        return new Redirect( $url );
    }
    public function json(array $data ) { // 怪しい
        return new Response( json_encode($data), Response::OK, [
            'Content-Type' => 'text/json',
        ] );
    }
}
```

簡単な使い方。

```php
$respond = new Respond();
$respond->view('template.file.name'); // view
$respond->redirect('http://www.example.com/test'); // redirect
$respond->json($jsonData); // json
```

コードはサンプルです。

### Requestとの関連性

しかし、現在のpathinfoからの相対パスで指定したいと思いませんか？
すると、こんなメソッドが必要でしょう。

```php
class Respond {
    public function to( $url ) {
        $url  = $this->request->getUriForPath( $url );
        return new Redirect( $url );
    }
}
```

このコードで動くかどうかはテストしないといけませんが、_リクエストオブジェクトが必要_ になることが分かります。

つまり、RedirectはRequestに依存することになります。
確かにリクエストに対してレスポンスを返すことを考えると、依存性は理解できます。

つまり

```php
class Respond {
    protected $request;
    public function __construct($request) {
        $this->request = $request;
    }
}
```

となります。

Request概念設計
--------------

Request側にリクエストごとに変わる変数を寄せる。
またレスポンスを返すのに必要なサービスを持つこととする。

今のところ、必要なサービスはResponderのみです。
ついでにセッションも設定してしまいましょう。

```php
use Symfony\Component\HttpFoundation\Request as SymfonyRequest;
use Symfony\Component\HttpFoundation\Session\Session;

class Request extends SymfonyRequest {
    public static function start( $storage = null ) {
        $request = new static( $_GET, $_POST, array(), $_COOKIE, $_FILES, $_SERVER );
        $request->setSession( new Session( $storage ) );
        return $request;
    }
}
```

テスト用にセッション用ストレージを指定できるようにしました。


### レスポンス生成

先のレスポンダーを設定できるようにします。簡単に。

```php
class Request extends SymfonyRequest {
    protected $respond;
    public function setRespond($respond) {
        $this->respond = $responder;
        return $this;
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
// このタイミングかな。
$request  = Request::start();
$request->setRespond( new Respond($request) );
```

他にRequestオブジェクトが必要なサービスは、この方法で作るか…

サービスが散らばるなぁ。

### コンテナが欲しい

だんだんコンテナが欲しくなってきた。

今後の設計で必要なことが増えてくると思います。


