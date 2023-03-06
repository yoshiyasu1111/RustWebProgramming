## ビューからデータを抽出する

このセクションでは、HTTPリクエストのヘッダーとボディからデータを抽出する方法について説明します。そして、これらの方法を使用して、ToDo項目を編集、削除し、ミドルウェアが完全にロードされる前にリクエストをインターセプトします。一度に一歩ずつ進んでいきます。とりあえず、ToDo項目を編集するためのHTTPリクエストのボディからデータを取り出してみましょう。JSON形式のデータを受け取る場合、この本でずっとやってきたように、このコードをビューから切り離すようにしましょう。考えてみれば、編集する項目を送信すればいいだけです。しかし、この同じスキーマを削除にも使うことができます。スキーマは、json_serialization/to_do_item.rsファイルに、以下のコードで定義することができます。

```rust
use serde::Deserialize;
#[derive(Deserialize)]
pub struct ToDoItem {
    pub title: String,
    pub status: String
}
```

JSONではenumを渡すことができず、文字列しか渡すことができないため、先のコードでは、各フィールドに必要なデータ型を記載しているだけである。JSONからのデシリアライズは、ToDoItem構造体をDeserialize traitマクロで装飾することで可能になります。ToDoItem構造体をアプリケーションの他の部分から利用できるようにすることを忘れてはいけませんので、json_serialization/mod.rsファイルは以下のようにします。

```rust
pub mod to_do_items;
pub mod to_do_item;
```

アイテムの抽出が終わったので、次はeditビューに移ります。views/to_do/edit.rsファイルでは、次のコードで必要なものをインポートすることができます。

```rust
use actix_web::{web, HttpResponse};
use serde_json::value::Value;
use serde_json::Map;
use crate::state::read_file;
use crate::to_do::{to_do_factory, enums::TaskStatus};
use crate::json_serialization::{to_do_item::ToDoItem, 
                                to_do_items::ToDoItems};
use crate::processes::process_input;
```

先のコードでは、ビューに必要な標準的なシリアライズとウェブ構造体をインポートする必要があることがわかります。また、データを取り込んでアプリケーションの状態全体を返すために、ToDoItemとToDoItemsという構造体をインポートしています。そして、コマンドで入力を処理するprocess_input関数をインポートすることができます。この時点で、インポートを見て、編集を行うために必要なステップを思い浮かべることができるでしょうか。前に進む前に考えてみてください。パスは、getビューで行ったのと同じです。しかし、新しく更新されたアイテムでステートを更新しなければなりません。また、editコマンドが渡された場合、process_input関数がToDo項目を編集することも覚えておく必要があります。

よく考えたら、問題を解決する方法はたくさんあることを思い出してください。自分の手順で問題が解決したのであれば、それが定められた手順と違っていても、悪い気はしないことです。より良い解決策が生まれるかもしれません。私たちの編集の仕方は、次のようなステップを踏んでいます。

1. ToDo項目に対してアプリケーション全体の状態を取得する。
2. アイテムがあるかどうかを確認し、ない場合はnot foundレスポンスを返す。
3. to_do_factoryファクトリーにデータを渡し、状態から既存のデータを操作できるアイテムに構築する。
4. 投入されるステータスが、既存のステータスと同じでないことをチェックする。
5. 既存のアイテムをeditコマンドでprocess_input関数に渡して、JSONステートファイルに保存されるようにする。
6. アプリケーションの状態を取得し、それを返す。

これらの手順を踏まえて、次のサブセクションでは、リクエストのボディからJSONを抽出し、編集のために処理する知識を具体化することができます。

### リクエストのボディからJSONを抽出する

インポートが完了し、アウトラインが定義されたので、次のコードでビューのアウトラインを定義することができます。

```rust
pub async fn edit(to_do_item: web::Json<ToDoItem>) -> HttpResponse {
    . . .
}
```

先のコードでは、ToDoItem構造体がweb::Json構造体にラップされていることがわかります。つまり、to_do_itemというパラメータは、リクエストのボディから抽出され、シリアライズされてToDoItem構造体として構築されます。つまり、ビューの内部では、to_do_itemはToDoItem構造体ということになります。このように、ビューの内部では、次のようなコードで状態をロードすることができます。

```rust
let state: Map<String, Value> = read_file("./state.json");
```

次に、以下のコードで、状態からアイテムデータを抽出することができます。

```rust
let status: TaskStatus;
match &state.get(&to_do_item.title) {
    Some(result) => {
        status = TaskStatus::new(result.as_str().unwrap());
    }
    None=> {
        return HttpResponse::NotFound().json(format!("{} not in state", &to_do_item.title))
    }
}
```

先のコードでは、データからステータスを構築したり、ステータスが見つからない場合は not found HTTP レスポンスを返したりできることがわかります。次に、次のコードを使用して、既存のデータでアイテム構造体を構築する必要があります。

```rust
let existing_item = to_do_factory(to_do_item.title.as_str(), status.clone());
```

先のコードでは、ファクトリーがなぜ便利なのかがわかります。ここで、アイテムの新しいステータスと既存のステータスを比較する必要があります。次のコードのように、希望するステータスが同じであれば、ステータスを変更する意味はないでしょう。

```rust
if &status.stringify() == &TaskStatus::from_string(
                        &to_do_item
                            .status
                            .as_str()
                            .to_string())
                        .stringify() {
    return HttpResponse::Ok().json(ToDoItems::get_state())
}
```

そのため、現在の状態を確認し、希望する状態と同じであれば、単にOkというHTTP応答状態を返す必要があります。このようにするのは、フロントエンドのクライアントが同期していない可能性があるからです。次の章では、フロントエンドのコードを書き、アイテムがキャッシュされレンダリングされることを確認します。例えば、アプリケーションで別のタブを開いていたり、電話などの別のデバイスでToDoアプリケーションを更新した場合、このリクエストを行うクライアントは同期していないかもしれません。同期していないフロントエンドに基づいてコマンドを実行することは避けたい。そこで、次のコードで編集を行い、状態を返すことで入力を処理する必要があります。

```rust
process_input(existing_item, "edit".to_owned(), &state);
return HttpResponse::Ok().json(ToDoItems::get_state())
```

前述のコードは動作するはずですが、現時点では動作しません。これは、TaskStatus enumのクローンを作成する必要があり、TaskStatusがClone traitを実装していないためです。これは、src/to_do/enums.rs ファイルで、次のコードで更新することができます。

```rust
#[derive(Clone)]
pub enum TaskStatus {
    DONE,
    PENDING
}
```

次に、編集ビューが利用可能で、To-Doビューファクトリーで定義されていることを確認する必要があります。そこで、src/views/to_do/mod.rsファイルの中で、私たちのファクトリーは次のようになるはずです。

```rust
mod create;
mod get;
mod edit;

use actix_web::web::{ServiceConfig, post, get, scope};

pub fn to_do_views_factory(app: &mut ServiceConfig) {
    app.service(
        scope("v1/item")
            .route("create/{title}", post().to(create::create))
            .route("get", get().to(get::get))
            .route("edit", post().to(edit::edit))
    );
}
```

ビューファクトリーがうまくスケーリングされていることがわかります。また、ToDo項目に関するすべてのビューが1つの独立したページでうまく定義されていることも評価できます。つまり、前のコードを見るだけで、削除ビューが必要であることを知ることができます。これでアプリケーションを実行し、Postmanで次のような設定を行い、リクエストを行うことができます。


図4.8 - Postmanでリクエストを編集する

図4.8を見ると、洗浄タスクを「DONE」に切り替えていることがわかります。このデータは、形式をJSONにして、rawとしてボディに入れました。これをeditエンドポイントに呼び出すと、次のようなレスポンスが返ってきます。

```json
{
    "pending_items": [
        {
            "title": "learn to code rust",
            "status": "PENDING"
        }
    ],
    "done_items": [
        {
            "title": "washing",
            "status": "DONE"
        }
    ],
    "pending_item_count": 1,
    "done_item_count": 1
}
```

先のコードでは、完了したアイテムのリストが入力され、カウントが変更されたことが確認できます。このまま同じ呼び出しを続けると、洗濯の項目がすでに完了した状態なのに、完了に編集することになり、同じ応答が返されます。洗濯のステータスを保留に戻すか、タイトルを変更するかして、異なる更新状態を取得する必要があります。ToDoItem構造体はこれら2つのフィールドを期待しているため、呼び出しの本文にタイトルとステータスを含めない場合は、即座に不正なリクエスト・レスポンスが返されることになります。

さて、URLのパラメータとボディでJSONデータを受け取り、返すというプロセスを押さえたので、ほぼ完了です。しかし、データ抽出に使用される重要な方法として、もう1つヘッダーを取り上げる必要があります。ヘッダーは、セキュリティ認証情報などのメタ情報を格納するために使用されます。

### リクエストのヘッダーからデータを抽出する

さまざまなリクエストを承認する必要がある場合、それらをすべてのJSON構造体に格納するのはスケーラブルではありません。また、リクエストボディが大きくなる可能性があることも認識する必要があります（特にリクエスト者が悪意を持っている場合）。したがって、リクエストをビューに渡す前に、セキュリティ認証情報にアクセスすることは理にかなっています。これは、一般的にミドルウェアと呼ばれるものを使ってリクエストをインターセプトすることで可能です。リクエストをインターセプトしたら、セキュリティ証明書にアクセスしてチェックし、ビューを処理することができます。

本書の前回では、認証のためのミドルウェアを手作業で開発しました。しかし、これではコード管理の面で拡張性に欠け、柔軟な対応ができません。しかし、独自のミドルウェアを手動で設定することで、サーバコンストラクタがどのように動作するかをより深く理解し、リクエストを柔軟に処理できるようにすることは重要です。リクエストをインターセプトするために、actix-service crateを追加する必要があります。このインストールにより、Cargo.tomlファイルの依存関係は以下のようになるはずです。

```toml
[dependencies]
actix-web = "4.0.1"
serde_json = "1.0.59"
serde = { version = "1.0.136", features = ["derive"] }
actix-service = "2.0.2"
```

それでは、src/main.rs ファイルを更新します。まず、importsは以下のようにします。

```rust
use actix_web::{App, HttpServer};
use actix_service::Service;
mod views;
mod to_do;
mod state;
mod processes;
mod json_serialization;
```

これですべてのインポートが完了したので、次のコードでサーバー構築を定義することができます。

```rust
#[actix_web::main]

async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let app = App::new()
            .wrap_fn(|req, srv| {
                println!("{:?}", req);
                let future = srv.call(req);
                async {
                    let result = future.await?;
                    Ok(result)
                }
        })
        .configure(views::views_factory);
        return app
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

先のコードでは、wrap_fnによってリクエスト(req)と対話することができることがわかります。ルーティングサービス(srv)は、リクエストを渡すために必要なときに呼び出すことができます。ルーティングサービスの呼び出しは、結果を返す非同期コードブロックの中で終了するのを待つ未来であることに注意しなければなりません。これはミドルウェアです。ルーティングサービスを呼び出してビューでHTTPリクエストを処理する前に、リクエストを操作し、検査し、ルートを変更したり返したりすることができます。私たちの場合は、リクエストのデバッグをプリントアウトするだけなので、次のようなプリントアウトになります。

```
ServiceRequest HTTP/1.1 GET:/v1/item/get
  headers:
    "accept-language": "en-GB,en-US;q=0.9,en;q=0.8"
    "accept": "text/html,application/xhtml+xml,application/xml;
    q=0.9,image/avif,image/webp,image/        apng,*/*;q=0.8,application
    signed-exchange;v=b3;q=0.9"
    "sec-ch-ua-platform": "\"macOS\""
    "sec-fetch-site": "none"
    . . . 
    "host": "127.0.0.1:8000"
    "connection": "keep-alive"
    "sec-fetch-user": "?1"
```

たくさんのデータを扱うことができることがわかります。しかし、自作のミドルウェアでは、ここまでが限界です。これからは、traitを使ってヘッダーからデータを抽出することを検討します。

### トレイトによるヘッダー抽出の簡略化

その前に、futures crateをインストールする必要があります。Cargo.tomlファイルのdependenciesセクションに以下を追加してください。

```
futures = "0.3.21"
```

ここで、JSON Web Token（JWT）を格納するためのsrc/jwt.rsファイルを作成することにします。このファイルには、ユーザーがログインした後に、ユーザーに関する暗号化されたデータを保存することができます。そして、すべてのHTTPリクエストでヘッダーにJWTを送信することができます。第7章「ユーザーセッションの管理」で、これらのトークンのチェックと管理に関するセキュリティ上のニュアンスを探ります。今のところ、ヘッダー抽出プロセスを単純に構築することにします。まず、以下のコードでsrc/jwt.rsファイルに必要な構造体とtraitをインポートすることから始めます。

```rust
use actix_web::dev::Payload;
use actix_web::{Error, FromRequest, HttpRequest};
use futures::future::{Ready, ok};
```

Payload構造体には、リクエストの生データストリームが格納されます。これは、ビューに表示される前にデータを抽出するために実装するものです。次に、Readyとok from futuresを使用して、データ抽出の結果をラップし、ヘッダー抽出の値である成功値ですぐにReadyになる未来を作成します。必要なものをインポートできたので、次のコードでJWT構造体を定義できます。

```rust
pub struct JwToken {
    pub message: String
}
```

今はメッセージだけですが、将来的にはユーザーのIDなどのフィールドを追加していく予定です。この構造体があれば、次のようなコードでFromRequest traitを実装することができます。

```rust
impl FromRequest for JwToken {
    type Error = Error;
    type Future = Ready<Result<JwToken, Error>>;
    fn from_request(req: &HttpRequest, _: &mut Payload) -> Self::Future {
    . . .
    }
}
```

ビューがロードされる前にfrom_request関数が呼び出されることが推測できます。私たちはヘッダを抽出しているため、ペイロードには興味がありません。そこで、パラメータを_でマークしています。これは、JwToken構造体またはエラーとなる結果を格納する準備のできた未来型です。from_request関数の内部では、次のコードでヘッダーからデータを抽出することができます。

```rust
match req.headers().get("token") {
    Some(data) => {
        let token = JwToken{
            message: data.to_str().unwrap().to_string()
        };
        ok(token)
    },
    None => {
        let token = JwToken{
            message: String::from("nothing found")
        };
        ok(token)
    }
}
```

先のコードで、この章では、トークン・キーを探し、それがあれば、メッセージとともにJwToken structを返すだけであることがわかる。なければ、何も見つからずにJwToken構造体を返します。しかし、第7章「ユーザーセッションの管理」では、この関数を再検討し、エラーをスローしたり、不正なコードでビューにヒットする前にリクエストを返したりするなどのコンセプトを探ります。とりあえず、JwToken構造体をsrc/main.rsファイルに以下のコード行で定義して、アクセスできるようにする必要があります。

```rust
mod jwt;
```

さて、せっかくtraitを実装したのだから、その使い方はコンパクトでシンプルにしましょう。views/to_do/edit.rsファイルのeditビューをもう一度見て、JwToken構造体をインポートし、editビューのパラメータにJwToken構造体を追加して、次のコードに見られるようにメッセージをプリントアウトしてみましょう。

```rust
use crate::jwt::JwToken;
pub async fn edit(to_do_item: web::Json<ToDoItem>, 
                  token: JwToken) -> HttpResponse {
    println!("here is the message in the token: {}", 
              token.message);
    . . .
```

明らかに、ビューの残りの部分を編集したくないのですが、先のコードから推測できるように、tokenパラメータはHTTPリクエストから抽出されたJwToken構造体で、ToDoItem構造体と同様にすぐに使える状態になっています。サーバを起動した後、同じ編集HTTPコールを行うと、HTTPリクエストが出力されるだけでなく、以下のようなレスポンスが得られることがわかります。

```
here is the message in the token: nothing found
```

まだヘッダーに何も追加していないので、動作しているように見えます。次の図で定義されたPostmanのセットアップで、ヘッダーにトークンを追加することができます。


図4.9 - Postmanのヘッダー付きリクエストの編集

再度リクエストを送信すると、以下のようなプリントアウトが得られます。

```
here is the message in the token: "hello from header"
```

これだけです！（笑ヘッダーを通してデータを渡すことができるのです。しかも、ビューからの追加や取り外しは、パラメータに定義して削除するのと同じくらい簡単です。
