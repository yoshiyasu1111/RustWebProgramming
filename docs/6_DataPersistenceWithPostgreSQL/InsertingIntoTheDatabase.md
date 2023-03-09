## データベースへの挿入

このセクションでは、ToDo項目を作成するビューを構築して作成します。ToDoアイテムの作成に関するルールを思い出すと、重複したToDoアイテムを作成したくありません。これは、一意制約で実現できます。しかし、今のところ、物事を単純にしておくのがよいでしょう。その代わりに、ビューに渡されるタイトルに基づくフィルタを使用して、データベースのフォールを作成します。そして、チェックし、結果が返ってこなければ、新しいToDo項目をデータベースに挿入することにします。これを行うには、views/to_do/create.rsファイルのコードをリファクタリングします。まず、次のコードにあるように、インポートを再設定します。

```rust
use crate::diesel;
use diesel::prelude::*;
use actix_web::HttpResponse;
use actix_web::HttpRequest;
use crate::json_serialization::to_do_items::ToDoItems;
use crate::database::establish_connection;
use crate::models::item::new_item::NewItem;
use crate::models::item::item::Item;
use crate::schema::to_do;
```

前のコードでは、前のセクションで説明したように、クエリを作成するために必要なディーゼルのインポートをインポートします。次に、ビューがリクエストを処理し、結果を定義するために必要なactix-web構造体をインポートします。そして、データベースとやり取りするためのデータベース構造体や関数をインポートします。これですべてが揃ったので、ビューの作成に取り掛かることができます。pub async fn create関数内では、まずリクエストからToDoアイテムのタイトルの参照を2つ取得します。

```rust
pub async fn create(req: HttpRequest) -> HttpResponse {
    let title: String = req.match_info().get("title"
    ).unwrap().to_string();
    . . .
}
```

先のコードでURLからタイトルを抽出したら、次のコードにあるように、データベース接続を確立し、その接続を使用してテーブルにデータベースコールを行うのです。

```rust
let connection = establish_connection();
let items = to_do::table
    .filter(to_do::columns::title.eq(&title.as_str()))
    .order(to_do::columns::id.asc())
    .load::<Item>(&connection)
    .unwrap();
```

前のコードを見てわかるように、クエリーは前のセクションのクエリーとほとんど同じです。ただし、タイトルカラムを参照するフィルターセクションがあり、タイトルと同じである必要があります。作成されるアイテムが本当に新しいものであれば、アイテムは作成されないので、結果の長さはゼロになります。続いて、長さがゼロの場合、次のコードに見られるように、NewItemデータモデルを作成し、それをデータベースに挿入して、関数の最後に状態を返すようにします。

```rust
if items.len() == 0 {
    let new_post = NewItem::new(title);
    let _ = diesel::insert_into(to_do::table).values(&new_post)
        .execute(&connection);
}
return HttpResponse::Ok().json(ToDoItems::get_state())
```

Dieselにはinsert関数があり、テーブルを受け取り、構築したデータモデルを参照していることがわかります。データベースストレージに切り替えたので、以下のインポートは不要なので削除します。

```rust
use serde_json::value::Value;
use serde_json::Map;
use crate::state::read_file;
use crate::processes::process_input;
use crate::to_do::{to_do_factory, enums::TaskStatus};
```

さて、このアプリを使って、ToDo項目を作成し、その項目がアプリケーションのフロントエンドにポップアップ表示されるのを確認することができます。このように、CreateとGet Stateの機能が動作し、データベースと連携していることが確認できます。問題がある場合、よくある間違いは docker-compose を起動するのを忘れてしまうことです。(注: そうしないと、アプリが起動していないため、データベースに接続することができません)この作業は忘れないようにしてください。しかし、ToDoアイテムのステータスをDONEに編集することはできません。これを行うには、データベース上のデータを編集する必要があります。

### データベースを編集する

データを編集する場合、データベースからデータモデルを取得し、Dieselからデータベース呼び出し関数でエントリーを編集することになります。編集関数をデータベースと連携させるために、views/to_do/edit.rsファイル内のビューを編集することができます。次のコードにあるように、インポートをリファクタリングすることから始めます。

```rust
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse};
use crate::json_serialization::{to_do_item::ToDoItem, 
                                to_do_items::ToDoItems};
use crate::jwt::JwToken;
use crate::database::establish_connection;
use crate::schema::to_do;
```

先のコードを見てわかるように、あるパターンが浮かび上がってきています。そのパターンとは、認証、データベース接続、スキーマを処理するための依存関係をインポートし、その後、望む処理を実行する関数を構築するというものです。インポートやその意味については、以前にも説明しました。今回の編集ビューでは、タイトルへの参照を1つだけ取得しなければなりませんが、これは次のコードで示されています。

```rust
pub async fn edit(to_do_item: web::Json<ToDoItem>, token: JwToken) -> HttpResponse {
    let connection = establish_connection();
    let results = to_do::table
        .filter(to_do::columns::title.eq(&to_do_item.title));
    let _ = diesel::update(results)
        .set(to_do::columns::status.eq("DONE"))
        .execute(&connection);

    return HttpResponse::Ok().json(ToDoItems::get_state())
}
```

先のコードでは、データをロードすることなく、データベースに対してフィルタを実行していることがわかります。つまり、結果変数がUpdateStatement構造体になっています。そして、このUpdateStatement構造体を使用して、データベースのアイテムをDONEに更新しています。JSONファイルを使用しなくなったので、views/to_do/edit.rsファイルの以下のインポートを削除することができます。

```rust
use crate::processes::process_input;
use crate::to_do::{to_do_factory, enums::TaskStatus};
use serde_json::value::Value;
use serde_json::Map;
use crate::state::read_file;
```

先のコードでは、update関数を呼び出し、データベースから取得した結果で満たしていることがわかります。そして、statusカラムをDoneに設定し、接続の参照を使って実行します。これで、ToDo項目を編集して、完了リストに移動させることができるようになりました。しかし、削除することはできません。これを行うには、最終的なエンドポイントをリファクタリングして、アプリがデータベースに接続されるように完全にリファクタリングする必要がありそうです。

### データを削除する

データの削除については、前節の編集時と同じアプローチをとります。データベースからアイテムを取得し、Dieselのdelete関数に渡して、その状態を返すのです。今はこのアプローチに慣れているはずなので、views/to_do/delete.rsファイルに自分で実装してみることをお勧めします。インポートについては、以下のようなコードを与えています。

```rust
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse};
use crate::database::establish_connection;
use crate::schema::to_do;
use crate::json_serialization::{to_do_item::ToDoItem, 
                                to_do_items::ToDoItems};
use crate::jwt::JwToken;
use crate::models::item::item::Item;
```

先のコードでは、Dieselのマクロを使えるように、Dieselのクレートとpreludeに依存しています。preludeがなければ、スキーマを使用することはできません。次に、クライアントにデータを返すために必要なActix Web構造体をインポートします。次に、ToDo項目データを管理するために構築したクレートをインポートします。delete関数については、次のようなコードになっています。

```rust
pub async fn delete(to_do_item: web::Json<ToDoItem>, token: JwToken) -> HttpResponse {
    let connection = establish_connection();
    let items = to_do::table
        .filter(to_do::columns::title.eq(&to_do_item.title.as_str()))
        .order(to_do::columns::id.asc())
        .load::<Item>(&connection)
        .unwrap();
    let _ = diesel::delete(&items[0]).execute(&connection);

    return HttpResponse::Ok().json(ToDoItems::get_state())
}
```

品質管理を行うために、次のようなステップを踏んでみましょう。

1. テキスト入力欄に「カヌーを買う」と入力し、「作成」ボタンをクリックしてください。
2. テキスト入力に「ドラゴンボートレースに行く」と入力し、「作成」ボタンをクリックします。
3. カヌーを買う」項目の編集ボタンをクリックします。このようにすると、フロントエンドで次のような出力が得られるはずです。

図6.6-期待される出力

先ほどの図では、カヌーは買ったものの、まだドラゴンボートレースには行っていません。そして、このアプリはPostgreSQLデータベースとシームレスに連動しています。ToDoアイテムの作成、編集、削除が可能です。前の章で定義した構造のおかげで、データベースのためのJSONファイル機構を切り捨てるのに、多くの作業を必要としませんでした。データを処理し、返すためのリクエストは、すでに用意されていたのです。今、アプリケーションを実行すると、コンパイル時に次のようなプリントアウトがあることにお気づきでしょうか。

```rust
warning: function is never used: `read_file`
  --> src/state.rs:17:8
   |
17 | pub fn read_file(file_name: &str) -> Map<String, Value> {
. . .
warning: function is never used: `process_pending`
  --> src/processes.rs:18:4
   |
18 | fn process_pending(item: Pending, command: String, 
                        state: &Map<String, Value>) {
```

先のプリントアウトによると、src/state.rsファイルやsrc/processes.rsファイルはもう使われていないようです。これらのファイルが使われなくなったのは、src/database.rsファイルが、実際のデータベースを使って、永続的なデータの保存、削除、編集を管理しているからです。また、Diesel QueryableおよびIdentifiable特性を実装したデータベース モデル構造体を使用して、データベースから構造体にデータを加工しています。したがって、src/state.rsとsrc/database.rsのファイルは不要になったので削除することができます。また、src/to_do/traitsモジュールも不要なので、こちらも削除できます。次に、traitsへの参照をすべて削除することができます。参照を削除するということは、src/to_do/mod.rsファイルからtraitを削除し、to_doモジュールのstructsでtraitのインポートと実装を削除することです。例えば、src/to_do/structs/pending.rsファイルからtraitを削除するには、以下のimportを削除すればよい。

```rust
use super::super::traits::get::Get;
use super::super::traits::edit::Edit;
use super::super::traits::create::Create;
```

そして、これらの特徴のうち、以下の実装を削除することができます。

```rust
impl Get for Pending {}
impl Edit for Pending {}
impl Create for Pending {}
```

また、src/to_do/structs/done.rsファイルのtraitを削除する必要があります。私たちは今、うまく構造化された孤立したコードの成果を評価する機会を得ました。いくつかのファイルとtraitを直接削除することで、既存のストレージメソッドを削除することができるので、永続的なストレージのためのさまざまなメソッドを簡単に組み込むことができるのです。そして、traitの実装を削除するだけです。これだけです。関数を調べたり、コード行を変更したりする必要はありませんでした。読み取りの機能はtraitの中にあったので、traitの実装を削除するだけで、アプリケーションを完全に切り離すことができました。このようにコードを構造化することは、将来的に本当に有益です。これまで、マイクロサービスのリファクタリングや、ストレージなどのメソッドのアップグレードや切り替えの際に、あるサーバーから別のサーバーにコードを引き抜かなければならなかったことがありました。分離されたコードを持つことで、リファクタリングがより簡単に、より早く、より間違いのないものになります。

私たちのアプリケーションは、データベースと完全に連動するようになりました。しかし、データベースの実装を改善することができます。しかし、その前に、次のセクションで設定コードをリファクタリングする必要があります。
