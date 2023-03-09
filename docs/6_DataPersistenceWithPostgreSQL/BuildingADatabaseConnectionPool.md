## データベース接続プールの構築

このセクションでは、データベース接続プールを作成します。データベース接続プールは、限られた数のデータベース接続です。アプリケーションがデータベース接続を必要とするとき、プールから接続を取り出し、アプリケーションが接続を必要としなくなったら、接続をプールに戻します。プールに接続が残っていない場合、アプリケーションは次の図に見られるように、利用可能な接続があるまで待ちます。


図6.7 データベース接続プール（接続数は3つまで

データベース接続をリファクタリングする前に、Cargo.tomlファイルに次の依存関係をインストールする必要があります。

```toml
lazy_static = "1.4.0"
```

次に、src/database.rsファイルに以下のコードを記述して、importを定義します。

```rust
use actix_web::dev::Payload;
use actix_web::error::ErrorServiceUnavailable;
use actix_web::{Error, FromRequest, HttpRequest};
use futures::future::{Ready, ok, err};
use lazy_static::lazy_static;
use diesel::{
   r2d2::{Pool, ConnectionManager, PooledConnection},
   pg::PgConnection,
};
use crate::config::Config;
```

先のコードで定義したインポートから、新しいデータベース接続を書き出すとどうなると思いますか？この時点で、立ち止まって、これらのインポートで何ができるかを考えるのは良い機会です。

インポートの最初のブロックは、リクエストがビューに到達する前にデータベース接続を確立するために使用されます。importの2番目のブロックは、データベース接続プールを定義することができます。最後の Config パラメータは、接続用のデータベース URL を取得するためのものです。インポートが完了したので、次のコードで接続プール構造体を定義することができます。

```rust
type PgPool = Pool<ConnectionManager<PgConnection>>;
pub struct DbConnection {
   pub db_connection: PgPool,
}
```

前のコードでは、PgPool構造体がプール内の接続を管理する接続マネージャであることを述べています。そして、次のコードで静的参照である接続を構築しています。

```rust
lazy_static! {
    pub static ref DBCONNECTION: DbConnection = {
        let connection_string = Config::new().map.get("DB_URL").unwrap().as_str().unwrap().to_string();
        DbConnection {
            db_connection: PgPool::builder()
                .max_size(8)
                .build(ConnectionManager::new(connection_string))
                .expect("failed to create db connection_pool")
        }
    };
}
```

前のコードでは、設定ファイルから URL を取得し、接続プールを構築しています。これは静的参照であるため、DBCONNECTION変数の寿命はサーバーの寿命と一致します。次のコードで、establish_connection関数をリファクタリングして、データベース接続プールから接続を取得することができます。

```rust
pub fn establish_connection() -> PooledConnection<ConnectionManager<PgConnection>> {
    return DBCONNECTION.db_connection.get().unwrap()
}
```

先のコードでは、PooledConnection構造体を返していることがわかります。しかし、establish_connection関数を必要なときに毎回呼び出すことはしたくありません。また、何らかの理由で接続を確立できない場合は、ビューに到達する前にHTTPリクエストを拒否するようにしたいです。JWToken構造体と同様に、データベース接続を作成し、そのデータベース接続をビューに渡す構造体を作成することができます。この構造体には1つのフィールドがあり、次のようにプールされた接続を指定します。

```rust
pub struct DB {
    pub connection: PooledConnection<ConnectionManager<PgConnection>>
}
```

このDB構造体を使えば、JWToken構造体と同様に、以下のコードでFromRequest traitを実装することができます。

```rust
impl FromRequest for DB {
    type Error = Error;
    type Future = Ready<Result<DB, Error>>;
    fn from_request(_: &HttpRequest, _: &mut Payload) -> Self::Future {
        match DBCONNECTION.db_connection.get() {
            Ok(connection) => {
                return ok(DB{connection})
            },
            Err(_) => {
                return err(ErrorServiceUnavailable(
                "could not make connection to database"))
            }
        }
    }
}
```

ここでは、データベース接続の取得を直接アンラップすることはしません。その代わり、接続時にエラーが発生した場合は、役立つメッセージとともにエラーを返します。接続が成功した場合は、その旨を返します。これをビューで実装することができます。コードの繰り返しを避けるため、ここでは編集ビューだけを使用することにしますが、このアプローチはすべてのビューに適用する必要があります。まず、以下のインポートを定義します。

```rust
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse};
use crate::json_serialization::{to_do_item::ToDoItem, 
                                to_do_items::ToDoItems};
use crate::jwt::JwToken;
use crate::schema::to_do;
use crate::database::DB;
```

先のコードでは、DB構造体をインポートしたことが確認できます。これで、編集ビューは次のようになります。


```rust
pub async fn edit(to_do_item: web::Json<ToDoItem>, 
    token: JwToken, db: DB) -> HttpResponse {
    let results = to_do::table.filter(to_do::columns::title
        .eq(&to_do_item.title));
    
    let _ = diesel::update(results)
        .set(to_do::columns::status.eq("DONE"))
        .execute(&db.connection);
    return HttpResponse::Ok().json(ToDoItems::get_state())
}
```

先のコードでは、DB構造体のconnectionフィールドを直接参照していることがわかります。実は、DB構造体を認証トークンのようにビューに渡せば、プールされた接続をビューに取り込むことができるのです。  
