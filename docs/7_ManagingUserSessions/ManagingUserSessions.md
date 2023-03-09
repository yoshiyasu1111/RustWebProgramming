## ユーザーセッションの管理

ユーザーに対しては、ログインできるようにする必要があります。つまり、ユーザーの認証情報をチェックするエンドポイントを作成し、レスポンスのヘッダを介してフロントエンドでユーザーに返すJWTを生成する必要があります。最初のステップは、src/json_serialization/login.rsファイルに以下のコードでログインスキーマを定義することです。

```rust
use serde::Deserialize;
#[derive(Deserialize)]
pub struct Login {
    pub username: String,
    pub password: String
}
```

これをsrc/json_serialization/mod.rsファイルのpub mod login;というコード行で登録することを忘れないようにしなければなりません。これが完了したら、ログインエンドポイントを構築します。第3章 HTTPリクエストの処理」の「Actix Webフレームワークを使用したビューの管理」で作成したsrc/views/auth/login.rsファイルを編集して、基本的なログインビューを宣言することで、これを実行できます。これは単に文字列を返すだけです。

さて、次のコードに示すように、必要なインポートを定義することで、このビューのリファクタリングを開始することができます。

```rust
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse, Responder};
use crate::database::DB;
use crate::models::user::user::User;
use crate::json_serialization::login::Login;
use crate::schema::users;
use crate::jwt::JwToken;
```

この段階で、インポートをちらっと見て、これから何をするのかがわかるようになっています。本文からユーザー名とパスワードを抽出するつもりです。次に、データベースに接続してユーザー名とパスワードを確認し、JwToken構造体を使用してユーザーに返すトークンを作成する予定です。同じファイル内の次のコードで、ビューのアウトラインを最初にレイアウトすることができます。

```rust
. . .
Use std::collections::HashMap;
pub async fn login(credentials: web::Json<Login>,
                   db: DB) -> impl HttpResponse {
    . . .
}
```

ここでは、受信リクエストのボディからログイン認証情報を受け取り、ビューの接続プールからデータベース接続を準備していることがわかります。そして、リクエストボディから必要な情報を抽出し、次のコードでデータベースを呼び出します。

```rust
let password = credentials.password.clone();
let users = users::table
    .filter(users::columns::username.eq(
        credentials.username.clone())
    ).load::<User>(&db.connection).unwrap();
```

さて、次のコードでデータベース呼び出しから期待したものが得られたかどうかを確認する必要があります。

```rust
if users.len() == 0 {
    return HttpResponse::NotFound().await.unwrap()
} else if users.len() > 1 {
    return HttpResponse::Conflict().await.unwrap()
}
```

ここでは、いくつかのアーリーリターンを行っています。ユーザーがいない場合は、not foundレスポンスコードを返します。これは、私たちが時々期待することです。しかし、そのユーザー名を持つユーザーが複数いる場合は、別のコードを返す必要があります。

表示されたユニークな制約のために、何かが非常に間違っています。将来、移行スクリプトがこれらのユニークな制約を元に戻すかもしれませんし、ユーザー・クエリが偶然に変更されるかもしれません。制約に反して破損したデータは、トラブルシューティングが困難な予期せぬ方法でアプリケーションを動作させる可能性があるため、これが起こったことをすぐに知る必要があります。

これで、正しい数のユーザーが取得されたことが確認できたので、次のように、インデックスゼロにある唯一のユーザーを確信を持って取得し、そのパスワードが通じるかどうかをチェックすることができます。

```rust
match users[0].verify(password) {
    true => {
        let token = JwToken::new(users[0].id);
        let raw_token = token.encode();
        let mut body = HashMap::new();
      body.insert("token", raw_token);
        HttpResponse::Ok().json(body)
    },
    false => HttpResponse::Unauthorized()
}
```

ここでは、verify関数を使用していることがわかります。パスワードが一致すれば、そのIDを使ってトークンを生成し、ボディでユーザーに返します。パスワードが正しくない場合は、代わりに不正なコードを返します。

ログアウトに関しては、より軽量なアプローチを取ろうと考えています。ログアウトビューで行うべきことは、2行のJavaScriptコードを実行することだけです。1つは、ローカルストレージからユーザートークンを削除し、ユーザーをメインビューに戻すことです。HTMLは、開くとすぐに実行されるJavaScriptをホストするだけでいいのです。したがって、src/views/auth/logout.rsファイルに以下のコードを記述することでこれを実現することができます。

```rust
use actix_web::HttpResponse;
pub async fn logout() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body("<html>\
                <script>\
                    localStorage.removeItem('user-token'); \
                    window.location.replace(
                        document.location.origin);\
                </script>\
              </html>")
}
```

このビューはすでに登録されているので、アプリを実行してPostmanで呼び出すことができます。


図7.7 - Postmanでログインエンドポイントを使用してアプリケーションにログインする

ユーザー名を変更すると404レスポンスコードになり、パスワードを変更すると401レスポンスコードになります。ユーザー名とパスワードが正しければ、図7.7に示すように、200レスポンスコードが得られ、ヘッダーのレスポンスにトークンが含まれることになります。しかし、レスポンスヘッダでトークンを使いたい場合は、token can't be decodedというメッセージが表示されます。次のセクションでは、認証要件を一掃することにします。
