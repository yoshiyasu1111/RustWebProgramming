## 認証要件の一掃

この章では、フロントエンドで認証処理を行う前に、 Rust サーバの認証処理を整理していきます。この章の流れを魅力的なものにするため、定期的に "ハウスキーピング "を行うことはしていません。では、To_doビューを更新していきます。まず、createビューを認証要件で更新することから始めます。そのためには、src/views/to_do/create.rsにあるcreateビューの関数シグネチャを、以下のようにします。

```rust
. . .
use crate::jwt::JwToken;
use crate::database::DB
pub async fn create(token: JwToken,
                    req: HttpRequest, db: DB) -> HttpResponse {
    . . .
```

また、トークンから得たIDで新しいアイテムを作成する際には、次のコードでユーザーIDを更新する必要があります。

```rust
if items.len() == 0 {
    let new_post = NewItem::new(title, token.user_id);
    let _ = diesel::
            insert_into(to_do::table).values(&new_post)
        .execute(&db.connection);
}
Return HttpResponse::Ok().json(
    ToDoItems::get_state(token.user_id)
)
```

deleteビューでは、リクエストを行ったユーザーに属するToDo項目を削除していることを確認する必要があります。ユーザーIDを使ったフィルターを追加しないと、ToDoアイテムの削除はランダムになってしまいます。このフィルターは、src/views/to_do/delete.rs ファイルに以下のコードで追加することができます。

```rust
. . .
Use crate::database::DB;
. . .
pub async fn delete(to_do_item: web::Json<ToDoItem>,
                    token: JwToken, db: DB) -> HttpResponse {
    let items = to_do::table
        .filter(to_do::columns::title.eq(
                    &to_do_item.title.as_str())
                )
        .filter(to_do::columns::user_id.eq(&token.user_id))
        .order(to_do::columns::id.asc())
        .load::<Item>(&db.connection)
        .unwrap();
    let _ = diesel::delete(&items[0]).execute(&db.connection);
    return HttpResponse::Ok().json(ToDoItems::get_state(
        token.user_id
    ))
}
```

データベースへの問い合わせを行う際に、フィルタ関数を単に連鎖させることができることがわかります。削除ビューで行ったことを考慮すると、src/views/to_do/edit.rsファイルにあるeditの認証要件をどのようにアップグレードするのでしょうか？この段階では、削除ビューのアップグレードと同じようなアプローチで、自分で編集ビューを更新してみることをお勧めします。この作業を行うと、あなたのエディットビューは次のようなコードになるはずです。

```rust
pub async fn edit(to_do_item: web::Json<ToDoItem>,
                  token: JwToken, db: DB) -> HttpResponse {
    let results = to_do::table.filter(to_do::columns::title
                              .eq(&to_do_item.title))
                              .filter(to_do::columns::user_
                                      id
                              .eq(&token.user_id));
    let _ = diesel::update(results)
        .set(to_do::columns::status.eq("DONE"))
        .execute(&db.connection);
    return HttpResponse::Ok().json(ToDoItems::get_state(
                                   token.user_id
    ))
}
```

これで特定のビューを更新できたので、次はgetビューに移ります。getビューには、他のすべてのビューに適用されるget_state関数もあります。src/views/to_do/get.rsにあるgetビューは、次のような形になっています。

```rust
use actix_web::Responder;
use crate::json_serialization::to_do_items::ToDoItems;
use crate::jwt::JwToken;
pub async fn get(token: JwToken) -> impl Responder {
    ToDoItems::get_state(token.user_id)
}
```

さて、先のコードに書かれていることは、驚くようなことではありません。ToDoItems::get_state関数にユーザーIDを渡していることがわかります。ToDoItems::get_state関数が実装されている場所、つまりすべてのToDoビューで、ユーザーIDを記入することを忘れてはいけません。次に、src/json_serialization/to_do_items.rsファイルのToDoItems::get_state関数を次のコードで再定義することができます。

```rust
. . .
use crate::database::DBCONNECTION;
. . .
impl ToDoItems {
    . . .
    pub fn get_state(user_id: i32) -> ToDoItems {
        let connection = DBCONNECTION.db_connection.get()
                         .unwrap();
        let items = to_do::table
                    .filter(to_do::columns::user_id.eq
                           (&user_id))
                    .order(to_do::columns::id.asc())
                    .load::<Item>(&connection)
                    .unwrap();
        let mut array_buffer = Vec::
                               with_capacity(items.len());
        for item in items {
            let status = TaskStatus::from_string(
            &item.status.as_str().to_string());
            let item = to_do_factory(&item.title, status);
            array_buffer.push(item);
        }
        return ToDoItems::new(array_buffer)
    }
}
```

ここでは、データベース接続とユーザーIDのフィルタを更新していることがわかります。これで、さまざまなユーザーに対応するためのコードが更新されました。もう1つ、変更しなければならないことがあります。Reactアプリケーションでフロントエンドのコードを書くことになるので、Reactの開発は本そのものなので、Reactのコーディングはできるだけシンプルにすることにします。Axiosを使ったヘッダ抽出やGET投稿など、フロントエンドの開発が複雑になりすぎないように、ログインにPostメソッドを追加し、ボディを使ってトークンを返すことにします。これは、これを実行するために必要なすべての概念をカバーしているため、自分で解決しようとする良い機会です。

この問題を自分で解決しようとした場合、次のようになるはずです。まず、src/json_serialization/login_response.rs ファイルに、以下のコードでレスポンス構造体を定義します。

```rust
use serde::Serialize;
#[derive(Serialize)]
pub struct LoginResponse {
    pub token: String
}
```

src/json_serialization/mod.rsファイルにpub mod login_responseを入れて、先行する構造体を宣言することを忘れないようにします。src/views/auth/login.rsにアクセスして、login関数に次のようなreturn文があることを確認します。

```rust
match users[0].clone().verify(credentials.password.clone()) {
    true => {
        let user_id = users[0].clone().id;
        let token = JwToken::new(user_id);
        let raw_token = token.encode();
        let response = LoginResponse{token:
                                     raw_token.clone()};
        let body = serde_json::
                   to_string(&response).unwrap();
        HttpResponse::Ok().append_header(("token",
                           raw_token)).json(&body)
    },
    false => HttpResponse::Unauthorized().finish()
}
```

備考

お気づきの方もいらっしゃるかもしれませんが、無断転載を少し変更し、以下のようにしました。

HttpResponse::Unauthorized().finish()

これは、ビュー関数の戻り値の型をHttpResponse構造体に変更したためで、次のような関数シグネチャになります。

(credentials: web::Json<Login>, db: DB) -> HttpResponse

json関数をレスポンスに追加すると、レスポンスがHttpResponseBuilderからHttpResponseに変わってしまうため、切り替えが必要だったのです。json 関数が呼び出されると、HttpResponseBuilder は使用できなくなります。オーサリングされていないレスポンスビルダーに戻ると、finish関数がHttpResponseBuilderをHttpResponseに変換していることが推測されます。また、以下のコードのように、awaitを使うことで、HttpResponseBuilderをHttpResponseに変換することができます。

HttpResponse::Unauthorized().await.unwrap()

ここでは、ヘッダーとボディでトークンを返していることがわかります。これによって、フロントエンドのコードを書くときに柔軟性と手軽さを得ることができます。しかし、これはベストプラクティスではないことを強調しておく必要があります。フロントエンドの開発セクションをシンプルに保つために、トークンをボディとヘッダーに戻すというアプローチを実施しています。次に、src/views/auth/mod.rsファイルにおいて、以下のコードでログインビューのPOSTメソッドを有効にすることができます。

```rust
mod login;
mod logout;
use actix_web::web::{ServiceConfig, get, post, scope};
pub fn auth_views_factory(app: &mut ServiceConfig) {
    app.service(
            scope("v1/auth")
            .route("login", get().to(login::login))
            .route("login", post().to(login::login))
            .route("logout", get().to(logout::logout))
    );
}
```

同じログインビューにget関数を積み重ねただけであることがわかります。これで、ログインビューでPOSTとGETが利用できるようになりました。次のセクションでは、認証トークンが期限切れになるように設定します。トークンの有効期限を設定することで、セキュリティが向上します。もしトークンが漏洩し、悪意ある人物がトークンを手に入れた場合、彼らはログインすることなく、好きなことを好きなだけできるようになります。しかし、トークンの有効期限が切れると、悪者はトークンが失効するまでの限られた時間しか使えなくなります。
