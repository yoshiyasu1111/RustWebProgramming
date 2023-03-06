## Actix Webフレームワークを使用したビューの管理

これまでのところ、main.rs ファイルですべてのビューを定義してきました。しかし、プロジェクトが大きくなるにつれ、この方法ではうまく拡張できなくなります。適切なビューを見つけるのは難しいですし、ビューを更新するとミスが発生する可能性があります。また、Webアプリケーションからモジュールを削除したり、Webアプリケーションにモジュールを挿入したりすることも難しくなります。また、すべてのビューを1つのページで定義する場合、より大きなチームがアプリケーションを開発する場合、多くのマージコンフリクトを引き起こす可能性があります。このため、ビューのロジックはモジュールに格納した方がよいでしょう。ここでは、認証を行うモジュールを作成することで、この問題を解決します。この章では、認証に関するロジックを構築することはありませんが、ビューモジュールの構造を管理する方法を検討する際に使用する、わかりやすい例として、このモジュールは最適です。コードを書く前に、このウェブアプリケーションは次のようなファイルレイアウトになっているはずです。

```
├── main.rs
└── views
    ├── auth
    │   ├── login.rs
    │   ├── logout.rs
    │   └── mod.rs
    ├── mod.rs
```

各ファイルの中のコードを説明すると、以下のようになります。

<dl>
    <dt>main.rs             </dt>    <dd>サーバーが定義されているエントリーポイント</dd>
    <dt>views/auth/login.rs </dt>    <dd>ログインするためのビューを定義するコード</dd>
    <dt>views/auth/logout.rs</dt>    <dd>ログアウトのためのビューを定義するコード</dd>
    <dt>views/auth/mod.rs   </dt>    <dd>auth用のビューを定義するファクトリー</dd>
    <dt>views/mod.rs        </dt>    <dd>アプリ全体のビューを定義するファクトリー</dd>
</dl>

まず、エントリーポイントとして、main.rsファイルに以下のコードを記述して、余計なものがない基本的なWebサーバを作成します。

```rust
use actix_web::{App, HttpServer};
mod views;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let app = App::new();
        return app
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

この前のコードはわかりやすく、驚くようなことはないはずです。後でコードを変更し、ビューの定義に移ることができます。この章では、ビューが何であるかを示す文字列を返したいだけです。これで、アプリケーションの構成がうまくいっていることがわかるでしょう。基本的なログインビューは、views/auth/login.rs ファイルに以下のコードで定義することができます。

```rust
pub async fn login() -> String {
    format!("Login view")
}
```

さて、views/auth/logout.rsファイルのlogoutビューが次のような形になっていることは、驚くことではないでしょう。

```rust
pub async fn logout() -> String {
    format!("Logout view")
}
```

ビューが定義されたので、あとはmod.rsファイルにファクトリーを定義して、サーバーがビューを提供できるようにする必要があります。ファクトリーは、次のような形でアプリのデータフローを提供します。

図3.4 - 私たちのアプリケーションのデータフロー

図 3.4 を見ると、ファクトリーを連結することで、多くの柔軟性が得られることがわかります。たとえば、アプリケーションからすべての認証ビューを削除したい場合、メインのビューファクトリーでコードを一行削除するだけで、これを実現することができます。また、モジュールを再利用することもできます。例えば、複数のサーバーでauthモジュールを使用する場合、auth viewsモジュールのgitサブモジュールを作成し、それを他のサーバーで使用することができます。authモジュールのファクトリービューは、views/auth/mod.rsファイルに以下のようなコードで構築することができます。

```rust
mod login;
mod logout;

use actix_web::web::{ServiceConfig, get, scope};

pub fn auth_views_factory(app: &mut ServiceConfig) {
    app.service(scope("v1/auth")
        .route("login", get().to(login::login))
        .route("logout", get().to(logout::logout))
    );
}
```

先のコードでは、ServiceConfig構造体のmutableな参照を渡していることがわかります。これにより、サーバーのビューなどを異なるフィールドに定義することができます。この構造体に関するドキュメントには、大規模なアプリケーションで設定を異なるファイルに分割できるようにするためと書かれています。次に、ServiceConfig構造体にサービスを適用します。このサービスによって、スコープで定義されたプレフィックスがすべて入力されるビューのブロックを定義することができます。また、ブラウザから簡単にアクセスできるようにするために、今のところgetメソッドを使用していることを明記しています。これで、views/mod.rsファイルのメインビューファクトリーに、以下のコードでauthビューファクトリーを差し込むことができます。

```rust
mod auth;

use auth::auth_views_factory;
use actix_web::web::ServiceConfig;

pub fn views_factory(app: &mut ServiceConfig) {
    auth_views_factory(app);
}
```

先のコードでは、たった1行のコードでviewsモジュール全体を切り分けることができました。また、モジュールを好きなだけ連鎖させることも可能です。例えば、authビューモジュールの中にサブモジュールを作りたい場合、authサブモジュールのファクトリーをauthファクトリーに送り込むだけでいいのです。また、1つのファクトリーで複数のサービスを定義することも可能です。main.rsは、configure関数が追加されただけで、ほぼ同じ内容になっています（以下のコードを参照）。

```rust
use actix_web::{App, HttpServer};

mod views;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let app = App::new().configure(views::views_factory);
        return app
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

App構造体でconfigure関数を呼び出すと、configure関数にviewsファクトリーが渡され、viewsファクトリーはconfig構造体をファクトリー関数に渡してくれる。configure関数はSelf（つまりApp構造体）を返すので、クロージャの最後に結果を返すことができます。これで、サーバーを実行すると、次のような結果になります。


図3.5 ログイン画面

期待通りのプレフィックスを持つアプリケーションが動作していることが確認できます。これで、自信を持ってHTTPリクエストを処理するための基本的なことはすべて網羅できました。
