## 基本的なActix Webサーバーの立ち上げ

Cargoを使ったビルドは簡単です。必要なのは、プロジェクトをビルドしたいディレクトリに移動して、以下のコマンドを実行することだけです。

```bash
$ cargo new web_app
```

先のコマンドで、基本的なCargo Rustプロジェクトが構築されます。このアプリケーションを探索すると、以下のような構成になります。

```
└── web_app
    ├── Cargo.toml
    └── src
         └── main.rs
```

これで、Cargo.tomlファイルに以下のコードでActix Webの依存関係を定義することができます。

```
[dependencies]
actix-web = "4.0.1"
```

先のコードの結果、Webアプリケーションの構築に移ることができます。とりあえず、次のコードでsrc/main.rsファイルにすべてを記述します。

```rust
use actix_web::{web, App, HttpServer, Responder, HttpRequest};

async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", name)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
            .route("/say/hello", web::get().to(|| async { "Hello Again!" }))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

先のコードでは、actix_web crateから必要な構造体とtraitをインポートしていることがわかります。ビューを定義するために、いくつかの異なる方法を使用していることがわかります。関数を構築してビューを定義しています。これは、HttpRequest構造体を取り込みます。そして、リクエストから名前を取得し、actix_webクレートのResponder traitを実装した変数を返します。レスポンダは、リクエストの型をHTTPレスポンスに変換します。.route("/", web::get().to(greet)) コマンドで、アプリケーションサーバー用に作成したこのgreet関数をルートビューとして割り当てています。また、.route("/{name}", web::get().to(greet)) コマンドで、URLから名前をgreet関数に渡すことができることも確認できました。最後に、クロージャを最終ルートに渡します。この構成で、次のコマンドを実行してみましょう。

```bash
$ cargo run
```

以下のようなプリントを得ることができます。

```
Finished dev [unoptimized + debuginfo] target(s) in 0.21s
 Running `target/debug/web_app`
```

先の出力で、今はロギングがないことがわかります。これは想定内のことで、後でロギングを設定することにします。さて、サーバーが稼動しているので、以下のURL入力のそれぞれについて、ブラウザに対応する出力が表示されるはずです。


- Input: http://127.0.0.1:8080/
- Output: Hello World!
- Input: http://127.0.0.1:8080/maxwell
- Output: Hello maxwell!
- Input: http://127.0.0.1:8080/say/hello
- Output: Hello Again!

src/main.rsファイルの前のコードでは、今まで出会ったことのない新しい構文があることがわかります。main関数を#[actix_web::main]マクロで装飾しています。これは、非同期メイン関数をActix Webシステムのエントリーポイントとしてマークするものです。これで、関数が非同期であることと、サーバを構築するためにクロージャを使用していることを確認できます。この2つのコンセプトは、次の数セクションで説明します。次のセクションでは、クロージャを調査して、何が起こっているのかを本当に理解することにします。
