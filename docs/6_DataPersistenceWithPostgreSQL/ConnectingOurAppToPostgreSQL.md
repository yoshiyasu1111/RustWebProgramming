## アプリをPostgreSQLに接続する

前節では、ターミナルを使ってPostgreSQLデータベースに接続することに成功しました。しかし、今後はToDoアイテムのデータベースへの読み書きを管理するアプリが必要になります。このセクションでは、Docker上で動作するデータベースにアプリケーションを接続します。接続するには、接続を確立し、それを返す関数を作成する必要があります。データベースの接続と設定を管理するには、もっと良い方法があることを強調しておきますが、それはこの章の最後で取り上げます。今のところ、アプリケーションを実行するために最も単純なデータベース接続を実装することにします。src/database.rs ファイルで、次のコードで関数を定義しています。

```rust
use diesel::prelude::*;
use diesel::pg::PgConnection;
use dotenv::dotenv;
use std::env;

pub fn establish_connection() -> PgConnection {
    dotenv().ok();
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    PgConnection::establish(&database_url)
        .unwrap_or_else(|_| panic!("Error connecting to {}", database_url))
}
```

先のコードでは、まず、diesel::prelude::*; importコマンドを使用していることにお気づきでしょうか。このコマンドは、connection、expression、query、serialization、result structsの範囲をインポートします。必要なインポートが完了したら、接続関数を定義します。まず、dotenv().();コマンドを使用して環境のロードに失敗しても、プログラムがエラーをスローしないようにする必要があります。

これが完了したら、環境変数からデータベースのURLを取得し、データベースのURLへの参照を使用して接続を確立します。正しいURLと正しいパラメータを使用していることを確認したいので、この作業を行わないとデータベースのURLが表示されなくなる可能性があります。接続は関数の最後のステートメントなので、これが返されることになります。

さて、独自の接続ができたので、スキーマを定義する必要があります。これは、ToDoアイテムの変数をデータ型にマッピングするものです。スキーマはsrc/schema.rsファイルの中で、以下のコードで定義することができます。

```rust
table! {
    to_do (id) {
        id -> Int4,
        title -> Varchar,
        status -> Varchar,
        date -> Timestamp,
    }
}
```

先のコードでは、テーブルが存在することを指定するディーゼルマクロ、table!このマップは簡単で、今後、データベースのクエリやインサートでカラムやテーブルを参照するためにこのスキーマを使用することになります。

データベース接続の構築とスキーマの定義ができたので、それらをsrc/main.rsファイル内で次のようにimportして宣言する必要があります。

```rust
#[macro_use] extern crate diesel;

extern crate dotenv;

use actix_web::{App, HttpServer};
use actix_service::Service;
use actix_cors::Cors;

mod schema;
mod database;
mod views;
mod to_do;
mod state;
mod processes;
mod json_serialization;
mod jwt;
```

先のコードでは、最初のインポートで手続き型マクロも有効にしています。もし、#[macro_use]タグを使用しなければ、他のファイルからスキーマを参照することができなくなります。また、スキーマ定義ではテーブルマクロを使用することができません。また、dotenvクレートもインポートしています。第5章「ブラウザでコンテンツを表示する」で作成したモジュールも残しておきます。また、スキーマとデータベースのモジュールも定義しています。これで、データモデルの構築を開始するのに必要なものがすべて揃いました。

### データモデルを作成する

Rustでは、データモデルを使用して、データベースからのデータに関するパラメータや動作を定義します。以下の図に示すように、データモデルはデータベースとRustアプリの間の橋渡し役として機能します。


図6.5 「モデル、スキーマ、データベースの関係

このセクションでは、ToDoアイテムのデータモデルを定義します。ただし、必要に応じて、アプリにデータモデルを追加できるようにする必要があります。そのために、以下のステップを実行します。

1. 新しいToDo項目データモデル構造体を定義する。
2. 次に、新しいToDoアイテム構造体のコンストラクタ関数を定義します。
3. そして最後に、ToDo項目のデータモデル構造体を定義します。

コードを書き始める前に、srcディレクトリに以下のようなファイル構造を定義しておきます。

```
├── models
│   ├── item
│   │   ├── item.rs
│   │   ├── mod.rs
│   │   └── new_item.rs
│   └── mod.rs
```

先の構造では、各データモデルがmodelsディレクトリの中にディレクトリを持っています。そのディレクトリの中に、モデルを定義するファイルが2つあります。一つは新規インサート用、もう一つはデータベース周りのデータを管理するためのものです。新規挿入データモデルには、IDフィールドがありません。

IDフィールドがないのは、データベースがアイテムにIDを割り当てるためで、事前に定義するわけではありません。しかし、データベースのアイテムを操作する際には、そのIDを取得し、IDでフィルタリングしたい場合があります。そのため、既存のデータ・アイテム・モデルには、IDフィールドが用意されています。新しいアイテム・データ・モデルは、new_item.rsファイルに次のようなコードで定義することができます。

```rust
use crate::schema::to_do;
use chrono::{NaiveDateTime, Utc};

#[derive(Insertable)]
#[table_name="to_do"]
pub struct NewItem {
    pub title: String,
    pub status: String,
    pub date: NaiveDateTime
}

impl NewItem {
    pub fn new(title: String) -> NewItem {
        let now = Utc::now().naive_local();
        return NewItem{
            title, 
            status: String::from("PENDING"), 
            date: now
        }
    }
}
```

前のコードでわかるように、テーブルの定義を参照するため、インポートしています。そして、タイトルとステータスを文字列として新しいアイテムを定義します。chronoクレートを使用して、日付フィールドをNaiveDateTimeとして定義しています。次に、ディーゼル マクロを使用して、この構造体に属するテーブルを「to_do」テーブルとして定義します。この定義には引用符が使われているので、騙されないでください。

スキーマをインポートしないと、参照を理解できないため、アプリはコンパイルされません。また、Insertable タグを使用してデータをデータベースに挿入できるようにすることを示す、別のディーゼル マクロを追加します。前述したように、この構造体はデータを挿入するためだけのものなので、このマクロにこれ以上タグを追加するつもりはありません。

また、新しい構造体の作成に関する標準的なルールを定義できるように、新しい関数が追加されました。たとえば、保留中の項目のみを新規作成するようにします。これにより、不正なステータスが作成されるリスクを軽減することができます。後で拡張する場合は、新しい関数でステータスの入力を受け入れ、それをマッチ ステートメントで実行して、ステータスが受け入れ可能なステータスのいずれかでない場合はエラーを投げるようにすることができます。また、日付フィールドがアイテムを作成した日付であることを自動的に表明します。

これを踏まえて、item.rsファイルに以下のコードでアイテムデータモデルを定義します。

```rust
use crate::schema::to_do;
use chrono::NaiveDateTime;
#[derive(Queryable, Identifiable)]
#[table_name="to_do"]
pub struct Item {
    pub id: i32,
    pub title: String,
    pub status: String,
    pub date: NaiveDateTime
}
```

先のコードでわかるように、NewItemとItemの構造体の違いは、コンストラクタ関数がないこと、InsertableタグをQueryableとIdentifiableに入れ替えたこと、そして構造体にidフィールドを追加したことだけです。これらをアプリケーションの他の部分で利用できるようにするために、models/item/mod.rsファイルに以下のコードで定義しています。

```rust
pub mod new_item;
pub mod item;
```

そして、models/mod.rsファイルでは、itemモジュールを他のモジュールに公開し、main.rsファイルでは、以下のコードを記述しています。

```rust
pub mod item;
```

次に、main.rsファイルの中で、次のようなコードでモデルのモジュールを定義します。

```rust
mod models;
```

これで、アプリ全体でデータモデルにアクセスできるようになりました。また、データベースへの読み書きに関する動作も固定化されました。次に、これらのデータモデルをインポートして、アプリで使用することにしましょう。

### データベースからデータを取得する

データベースと対話するとき、その方法に慣れるまで時間がかかることがあります。オブジェクトリレーショナルマッパー（ORM）や言語が異なれば、その癖も異なります。基本的な原理は同じですが、これらのORMの構文は大きく異なることがあります。したがって、ORMを使用して時間を計るだけで、より自信を持ち、より複雑な問題を解決できるようになるのです。まずは、テーブルからすべてのデータを取得するという、最もシンプルな仕組みから始めてみましょう。

これを調べるには、to_doテーブルからすべての項目を取得し、各ビューの最後にそれを返すようにします。この仕組みは、第4章「HTTPリクエストの処理」で定義しました。今回のアプリケーションでは、フロントエンドのToDoアイテムをパッケージ化するget_state関数があることを思い出してください。このget_state関数は、src/json_serialization/to_do_items.rsファイル内のToDoItems構造体に格納されています。最初に、以下のコードで必要なものをインポートする必要があります。

```rust
use crate::diesel;
use diesel::prelude::*;
use crate::database::establish_connection;
use crate::models::item::item::Item;
use crate::schema::to_do;
```

先のコードでは、データベースクエリを構築できるディーゼルクレートとマクロをインポートしています。次に、データベースへの接続を行うためにestablish_connection関数をインポートし、クエリを作成しデータを処理するためにスキーマとデータベースモデルをインポートします。次に、get_state関数を次のようなコードでリファクタリングすることができます。

```rust
pub fn get_state() -> ToDoItems {
    let connection = establish_connection();
    let mut array_buffer = Vec::new();
    let items = to_do::table
        .order(to_do::columns::id.asc())
        .load::<Item>(&connection).unwrap();
    for item in items {
        let status = TaskStatus::new(&item.status.as_str());
        let item = to_do_factory(&item.title, status);
        array_buffer.push(item);
    }                            
    return ToDoItems::new(array_buffer)
}
```

先のコードでは、まず接続を確立しています。データベース接続が確立されると、次にテーブルを取得し、データベースクエリを構築します。クエリの最初の部分は、順序を定義しています。見てわかるように、このテーブルでは、独自の関数を持つカラムへの参照も渡すことができます。

そして、データのロードに使用する構造体を定義し、その接続への参照を渡します。マクロで構造体を定義しているので、NewItem構造体をload関数に渡すと、その構造体ではQueryableマクロが有効でないため、エラーが発生します。

また、load関数では、load::<Item>(&connection)でパラメータが渡される関数の括弧の前に接頭辞が付いていることにも注目できます。この接頭辞の考え方は、第1章「Rustクイック入門」の「traitsによる検証」の項、ステップ4で説明しましたが、とりあえず、以下のコードを参考にしてください。

```rust
fn some_function<T: SomeType>(some_input: &T) -> () {
    . . .
}
```

接頭辞<Item>に渡される型は、受け入れる入力の型を定義する。つまり、コンパイラが実行されると、ロード関数に渡された型ごとに別々の関数がコンパイルされることになります。  

次に、load関数を直接アンラップして、データベースからアイテムのベクトルを取得します。データベースからのデータで、アイテム構造体を構築してバッファに追加するループを実行します。これが完了したら、バッファからToDoItems JSONスキーマを構築し、それを返します。この変更により、すべてのビューがデータベースから直接データを返すようになりました。これを実行すると、表示されるアイテムは何もありません。また、アイテムを作成しようとしても、表示されません。しかし、表示されないとはいえ、データベースからデータを取得し、必要なJSON構造でシリアライズしているのです。これは、データベースからデータを返し、それを標準的な方法で要求元に返すという基本です。これは、Rustで作られたAPIのバックボーンです。データベースからアイテムを取得する際に、JSON状態ファイルからの読み込みに依存しなくなったので、src/json_serialization/to_do_items.rsファイルから以下のインポートを削除することができます。

```rust
use crate::state::read_file;
use serde_json::value::Value;
use serde_json::Map;
```

他のエンドポイントをリファクタリングしていないため、先行するインポートを削除しています。その結果、createエンドポイントは正しく起動しますが、このエンドポイントは、return_stateがもはや読み取らないJSONステートファイル内のアイテムを作成するだけです。再び作成を可能にするためには、データベースに新しいアイテムを挿入するために、createエンドポイントをリファクタリングする必要があります。
