## JSONシリアライズのためのマクロの使用

データをシリアライズしてクライアントに返す場合、Actix-web crateのJSONを使用すれば、最小限のコードで素早くこれを実現することができます。views/to_do/get.rsファイルに、すべてのToDoアイテムを返すGETビューを作成することで、これを実証できます。

```rust
use actix_web::{web, Responder};
use serde_json::value::Value;
use serde_json::Map;
use crate::state::read_file;

pub async fn get() -> impl Responder {
    let state: Map<String, Value> = read_file("./state.json");
    return web::Json(state);
}
```

ここでは、JSONファイルからJSONを読み込み、これをweb::Json関数でラップして値を返しているに過ぎないことがわかります。JSONファイルからMap<String, Value>を直接返すだけなら、StringとValueということで理にかなっているかもしれません。しかし、Map<String, Value> の型は、レスポンダの特性を実装していません。以下のコードで、状態を直接返すように関数を更新することができます。

```rust
pub async fn get() -> Map<String, Value>  {
    let state: Map<String, Value> = read_file("./state.json");
    return state;
}
```

しかし、views/to_do/mod.rs ファイルの get().to() 関数は、レスポンダの特性を実装した struct を受け入れる必要があるため、これは機能しません。そこで、views/to_do/mod.rs ファイルに以下のコードで get ビューを挿入することができるようになりました。

```rust
mod create;
mod get; // import the get file 

use actix_web::web::{ServiceConfig, post, get, scope};
// import get
pub fn to_do_views_factory(app: &mut ServiceConfig) {
    app.service(
        scope("v1/item")
            .route("create/{title}", post().to(create::create))
            .route("get", get().to(get::get)) // define view and URL
    );
}
```

URL http://127.0.0.1:8000/item/get を実行すると、レスポンスボディに以下のような JSON データが得られます。

```json
{
    "learn to code rust": "PENDING",
    "washing": "PENDING"
}
```

これで、フロントエンドに提示できる構造化データを手に入れることができました。これは基本的に仕事を完了させるものですが、あまり便利ではありません。例えば、保留と完了の2種類のリストを用意したい。また、ToDoがいつ作成されたか、編集されたかを示すタイムスタンプを追加することができます。ToDoアイテムのタイトルとステータスを返すだけでは、必要なときに複雑さを拡張することはできません。

### 独自のシリアライズ構造体を構築する

ユーザーに返すデータの種類をより細かく制御するために、独自のシリアライズ構造体を構築する必要があります。このシリアライズ構造体では、完了したアイテムと保留中のアイテムの2つのリストを表示する予定です。リストには、タイトルとステータスで構成されるオブジェクトが配置されます。第2章「RustでWebアプリケーションを設計する」を思い出すと、保留アイテム構造体と完了アイテム構造体は、Base構造体から合成によって継承されます。したがって、Base構造体からタイトルとステータスにアクセスする必要があります。しかし、このBase構造体は一般には公開されていません。各ToDoアイテムの属性をシリアライズするために、アクセスできるようにする必要があります。


図4.7 - ToDo構造体とインターフェースの関係

図4.7を見ると、TaskStatus列挙型が依存関係の根源であることがわかります。ToDo アイテムをシリアライズする前に、この enum をシリアライズできるようにする必要があります。これには、serde クレートが使用できます。これを行うには、Cargo.toml ファイルで依存関係を更新する必要があります。

```toml
[dependencies]
actix-web = "4.0.1"
serde_json = "1.0.59"
serde = { version = "1.0.136", features = ["derive"] }
```

features = ["derive"]を追加したことがわかります。これにより、構造体をserde traitsで装飾することができるようになります。次に、src/to_do/enums.rsファイルにおいて、以下のコードでenumをどのように定義したかを見てみましょう。

```rust
pub enum TaskStatus {
    DONE,
    PENDING
}
impl TaskStatus {
    pub fn stringify(&self) -> String {
        match &self {
            &Self::DONE    => { return "DONE".to_string()    },
            &Self::PENDING => { return "PENDING".to_string() }
        }
    }
    pub fn from_string(input_string: String) -> Self {
        match input_string.as_str() {
            "DONE"    => TaskStatus::DONE,
            "PENDING" => TaskStatus::PENDING,
            _         => panic!("input {} not supported", input_string)
        }
    }
}
```

先のコードでは、DONEとPENDINGという2つのフィールドがあることがわかります。しかし、これらは本質的に独自の型です。これをJSONの値としてシリアライズするにはどうすればよいのでしょうか。stringify関数にヒントがあります。しかし、これは全貌を表しているわけではありません。サーバビューの戻り値は、traitを実装する必要があることを思い出してください。src/to_do/enums.rsファイルに必要なtraitを以下のコードでインポートすれば、TaskStatus列挙型のserde traitを実装することができます。

```rust
use serde::ser::{Serialize, Serializer, SerializeStruct};
```

これでSerialize traitを実装するのに必要なものはすべて揃いましたので、次の節で書いた構造体をどのようにシリアライズするかをカスタマイズすることができます。

### Serialize トレイトの実装

Serialize はこれから実装する trait で、Serializer は serde がサポートしているあらゆるデータ形式をシリアライズできるデータフォーマッターです。そして、以下のコードで TaskStatus enum に対して Serialize trait を実装することができます。

```rust
impl Serialize for TaskStatus {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        Ok(serializer.serialize_str(&self.stringify().as_str())?)
    }
}
```

これは、serdeのドキュメントで定義されている標準的な方法です。先のコードでは、serialize関数が定義されていることがわかります。このserialize関数は、TaskStatus列挙型をシリアライズする際に呼び出されます。また、serializerの記法がSであることもわかります。そこで、SをSerializerと定義するwhere文を使用します。これは直感に反すると思われるかもしれませんので、アプリケーションから一歩下がって調べてみることにしましょう。次のコードブロックは、このアプリケーションを完成させるために必要なものではありません。

ここで、基本的な構造体を以下のように定義しておきます。

```rust
#[derive(Debug)]
struct TwoDposition {
    x: i32,
    y: i32
}
#[derive(Debug)]
struct ThreeDposition {
    x: i32,
    y: i32,
    z: i32
}
```

先のコードでは、TwoDpositionとThreeDpositionの両方の構造体に対してDebug traitを実装していることがわかります。そして、各構造体に対してデバッグ文を出力する関数を、以下のコードで定義することができます。

```rust
fn print_two(s: &TwoDposition) {
    println!("{:?}", s);
}
fn print_three(s: &ThreeDposition) {
    println!("{:?}", s);
}
```

しかし、これではうまく拡張できないことがわかります。この関数を実装しているものすべてに対して関数を書くことになります。代わりに、whereステートメントを使用して、Debug traitを実装している両方の構造体をwhereステートメントに渡すことができるようにします。まず、次のようなコードでtraitをインポートする必要があります。

```rust
use core::fmt::Debug;
```

そして、次のようなコードでフレキシブルな関数を定義することができます。

```rust
fn print_debug<S>(s: &S)
where
    S: Debug {
    println!("{:?}", s);    
}
```

ここで起こっているのは、関数に渡す変数の型が汎用的であるということです。つまり、SはDebug特性を実装していれば、どのような型でもよいということです。もしDebug traitを実装していない構造体を渡そうとすると、コンパイラはコンパイルを拒否します。では、コンパイルするとどうなるのでしょうか？次のコードを実行してみてください。

```rust
fn main() {
    let two = TwoDposition{x: 1, y: 2};
    let three = ThreeDposition{x: 1, y: 2, z: 3};    
    print_debug(&two);
    print_debug(&three);
}
```

以下のようなプリントを得ることができます。

```
TwoDposition { x: 1, y: 2 }
ThreeDposition { x: 1, y: 2, z: 3 }
```

先の出力は、debug traitを起動したときの印刷結果なので、意味があります。しかし、これらはコンパイラがコンパイルする際に作成される2つの異なる関数です。私たちのコンパイラは、次の2つの関数をコンパイルしました。

```rust
print_debug::<TwoDposition>(&two);
print_debug::<ThreeDposition>(&three);
```

これは、Rustの仕組みとして知っていることを壊すものではありませんが、コードのスケーラビリティを高めるものです。where文を使う利点は他にもあります。例えば、次のようなコードでイテレータに必要な特性を指定することができます。

```rust
fn debug_iter<I>(iter: I)
where
    I: Iterator
    I::Item: Debug
{
    for item in iter {
        println!("{:?}", iter);
    }
}
```

先のコードでは、イテレータを受け取っていること、イテレータ内のアイテムがDebug traitを実装する必要があることがわかります。しかし、traitの実装を探求し続けると、本書の主な目的である「RustによるWebプログラミング」に集中できなくなる可能性があります。  

where文を使ってtraitを実装する知識を得て、TaskStatus enumのSerialize traitの実装を振り返ることができます。

```rust
impl Serialize for TaskStatus {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        Ok(serializer.serialize_str(&self.stringify().as_str())?)
    }
}
```

単にstringify関数を呼び出して、Okという結果で包んでいることがわかります。私たちは、より大きなデータ本体に挿入するため、ステータスをStringにしたいだけなのです。もしこれがフィールドを持つ構造体であれば、serialize関数を次のように書くことができます。

```rust
fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    let mut s = serializer.serialize_struct("TaskStatus", 1)?;
    s.serialize_field("status", &self.stringify())?;
    s.end()
}
```

先のコードでは、シリアライザは「TaskStatus」という構造体で、フィールド数は1です。そして、stringify関数の結果をstatusフィールドに帰着させる。こうすることで、基本的に以下のような構造体になります。

```rust
#[derive(Serialize)]
struct TaskStatus {
    status: String
}
```

しかし、今回の演習では、ステータスをより大きなボディに挿入して返す必要があるため、serialize_structのアプローチは利用しないことにします。

### シリアライズ構造体をアプリケーションコードに組み込む

TaskStatus列挙型のシリアライズが可能になったので、図4.7を振り返って、次にシリアライズされるのはBase構造体であることを確認します。また、Base構造体がJSONシリアライズの鍵であることがわかりますが、現在は公開されていないため、公開する必要があることがわかります。これは、to_do/structs/mod.rsファイルのBaseモジュールの宣言をmod base;からpub mod base;に変更することで実現できます。これで、Base構造体がモジュールの外で直接利用できるようになったので、srcディレクトリに以下のような構造のjson_serializationモジュールを自作することができます。

```
├── main.rs
├── json_serialization
│   ├── mod.rs
│   └── to_do_items.rs
```

getビューが呼ばれたときにビューアに返すものを、src/json_serialization/to_do_items.rsファイルに以下のコードで定義することにします。

```rust
use serde::Serialize;
use crate::to_do::ItemTypes;
use crate::to_do::structs::base::Base;
#[derive(Serialize)]
pub struct ToDoItems {
    pub pending_items: Vec<Base>,
    pub done_items: Vec<Base>,
    pub pending_item_count: i8,
    pub done_item_count: i8
}
```

先のコードでは、標準的なpublic structのパラメータを定義しただけです。次に、deriveマクロを使用してSerialize traitを実装しました。これにより、構造体の属性は、属性名をキーとしてJSONにシリアライズされるようになります。たとえば、ToDoItems構造体のdone_item_countが1だった場合、JSON本体は「done_item_count」と表記されます。1.この方法は、先ほどTaskStatus列挙型に対して行った手動によるシリアライズよりも簡単であることがわかります。これは、フィールドのフォーマットが単純であるためです。シリアライズ時に余計なロジックが必要ない場合は、ToDoItemsをSerialize traitで装飾するのが最も簡単で、結果的にエラーも少なくなります。  

さて、シリアライズが定義されたところで、データの処理について考えなければなりません。構造体を呼び出す前に、データをソートしてカウントしなければならないとしたら、スケーラブルとは言えません。この場合、当該ビューに属するロジックとは別に、シリアライズのためにデータを処理するビューに不要なコードが追加されることになります。また、重複するコードも発生します。データのソート、カウント、シリアライズを行う方法は1つだけです。もし、他のビューがアイテムのリストを返すために必要であれば、またコードを複製しなければならないでしょう。

このことを考えると、ToDo項目のベクトルを取り込み、適切な属性にソートし、それをカウントする構造体のコンストラクタを作成することは理にかなっていると言えます。このコンストラクタは、次のようなコードで定義することができます。

```rust
impl ToDoItems {
    pub fn new(input_items: Vec<ItemTypes>) -> ToDoItems {
        . . . // code to be filled in
    }
}
```

前のコードでは、コンストラクタがJSONファイルから読み込んだToDoアイテムのベクトルを受け取っていることがわかります。コンストラクタの内部では、次のステップを実行する必要があります。

1. アイテムを2つのベクトルにソートします。1つは保留アイテム、もう1つは完了アイテムです。

以下のコードで、アイテムの種類に応じて異なるベクターに追加して、アイテムのベクターをループするだけである。

```rust
let mut pending_array_buffer = Vec::new();
let mut done_array_buffer = Vec::new();
for item in input_items {
    match item {
        ItemTypes::Pending(packed)  => pending_array_buffer.push(packed.super_struct),
        ItemTypes::Done(packed)     => done_array_buffer.push(packed.super_struct)
    }
}
```

2. 保留と完了の合計数をカウントします。

次のステップでは、各ベクトルに対してlen関数を呼び出します。len関数は、ポインタサイズの符号なし整数型であるusizeを返します。このため、次のコードでi8としてキャストすることができます。

```rust
let done_count: i8 = done_array_buffer.len() as i8;
let pending_count: i8 = pending_array_buffer.len() as i8;
```

3. これで、構造体の構築と返却に必要なデータがすべて揃いました。構造体は、次のコードで定義できます。

```rust
return ToDoItems{
    pending_items:      pending_array_buffer, 
    done_item_count:    done_count,
    pending_item_count: pending_count, 
    done_items:         done_array_buffer
}
```

これでコンストラクタは完成です。

これで、この関数を使用して構造体を構築できるようになりました。あとは、これをアプリケーションにプラグインして、アプリケーションに渡せるようにするだけです。json_serialization/mod.rsファイルの中で、次のコードでpublicにすることができます。

```rust
pub mod to_do_items;
```

これで、src/main.rsファイルの中で、次のコードでモジュールを宣言することができます。

```rust
mod json_serialization;
```

また、src/to_do/structs/mod.rsファイルでは、ベースモジュールがpublicであることを確認する必要があります。src/to_do/structs/base.rsファイルでは、次のようなコードで、データを返すときに｜構造体をシリアライズすることにしています。

```rust
pub mod to_do_items;
use super::super::enums::TaskStatus;
use serde::Serialize;
#[derive(Serialize)]
pub struct Base {
    pub title: String,
    pub status: TaskStatus
}
```

構造体を利用するには、views/to_do/get.rsファイルのGETビューで定義し、以下のコードで構造体を返す必要があります。

```rust
use actix_web::{web, Responder};
use serde_json::value::Value;
use serde_json::Map;
use crate::state::read_file;
use crate::to_do::{ItemTypes, to_do_factory, enums::TaskStatus};
use crate::json_serialization::to_do_items::ToDoItems;

pub async fn get() -> impl Responder {
    let state: Map<String, Value> = read_file("./state.json");
    let mut array_buffer = Vec::new();

    for (key, value) in state {
        let status = TaskStatus::from_string(&value.as_str().unwrap()).to_string();
        let item: ItemTypes = to_do_factory(&key, status);
        array_buffer.push(item);
    }
    let return_package: ToDoItems = ToDoItems::new(array_buffer);

    return web::Json(return_package);
}
```

先のコードも、すべてがカチッとはまった瞬間の例です。read_fileインターフェースを使用して、JSONファイルからステートを取得します。そして、マップをループしてアイテムタイプを文字列に変換し、to_do_factoryインターフェイスに送り込みます。ファクトリーから構築されたアイテムを取得したら、それをベクターに追加して、そのベクターをJSONシリアライズ構造体に送り込みます。getビューを実行すると、次のようなJSONボディを受け取ります。

```json
{
    "pending_items": [
        {
            "title": "learn to code rust",
            "status": "PENDING"
        },
        {
            "title": "washing",
            "status": "PENDING"
        }
    ],
    "done_items": [],
    "pending_item_count": 2,
    "done_item_count": 0
}
```

これで、拡張や編集が可能な、構造化された応答ができました。アプリケーションの開発は決して止まらないので、このアプリケーションを維持し続けるつもりなら、この戻り値のJSONボディに機能を追加していくことになるでしょう。私たちはすぐに他のビューに移動します。しかし、その前に、APIコールを行うたびに、カウントを含むアイテムの全リストを返すことになることを認識しなければなりません。そうでなければ、他のすべてのビューで、getビューで書いたのと同じコードを書くことになります。次のセクションでは、ToDo項目を複数のビューで返せるようにパッケージ化する方法について説明します。

### ユーザーに返却するカスタムシリアル化された構造体をパッケージングする。

さて、GET ビューは Responder trait の実装を返します。つまり、ToDoItems構造体もこれを実装していれば、ビューで直接返すことができるのです。これは、json_serialization/to_do_items.rs ファイルで実現できます。まず、以下の構造体とtraitをインポートする必要があります。

```rust
use serde::Serialize;
use std::vec::Vec;
use serde_json::value::Value;
use serde_json::Map;
use actix_web::{
    body::BoxBody, http::header::ContentType, 
    HttpRequest, HttpResponse, Responder,
};
use crate::to_do::ItemTypes;
use crate::to_do::structs::base::Base;
use crate::state::read_file;
use crate::to_do::{to_do_factory, enums::TaskStatus};
```

actix_webクレートから、HTTPレスポンスを構築するためのさまざまな構造体やtraitをインポートしていることがわかります。ToDoItems構造体のget_state関数に、以下のコードでget viewコードを実装できるようになりました。

```rust
impl ToDoItems {
    pub fn new(input_items: Vec<ItemTypes>) -> ToDoItems {
        . . .
    }
    pub fn get_state() -> ToDoItems {
        let state: Map<String, Value> = read_file("./state.json");
        let mut array_buffer = Vec::new();
        for (key, value) in state {
            let status = TaskStatus::from_string(&value.as_str().unwrap().to_string());
            let item = to_do_factory(&key, status);
            array_buffer.push(item);
        }
        return ToDoItems::new(array_buffer)
    }
}
```

このように、1行のコードでJSONファイルからすべてのToDo項目を取得することができます。このToDoItems構造体をビューで返せるようにするには、以下のコードでResponder traitを実装する必要があります。

```rust
impl Responder for ToDoItems {
    type Body = BoxBody;
    fn respond_to(self, _req: &HttpRequest) -> HttpResponse<Self::Body> {
        let body = serde_json::to_string(&self).unwrap();
        HttpResponse::Ok()
            .content_type(ContentType::json())
            .body(body)
    }
}
```

前述のコードでは、serde_jsonクレートを使用してToDoItems構造体をシリアライズし、ToDoItems構造体をボディに持つHTTPレスポンスを返すということを本質的に行っています。respond_to関数は、ToDoItems構造体がビューで返されるときに呼び出されます。さて、ここからが本番です。views/to_do/get.rsファイルを次のようなコードに書き換えることができます。

```rust
use actix_web::Responder;
use crate::json_serialization::to_do_items::ToDoItems;
pub async fn get() -> impl Responder {
    return ToDoItems::get_state();
}
```

これでいい！今、アプリケーションを実行すると、以前と同じ応答が得られます。このように、traitはビューのコードを抽象化することができることがわかります。さて、getビューを作ったので、次はcreate、edit、deleteのビューを作ることにしましょう。そのために、次のセクション、つまりビューからデータを抽出することに進みます。
