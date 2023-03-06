## 環境との相互作用

環境と対話するためには、2つのことを管理する必要があります。まず、ToDoアイテムの状態をロード、保存、編集する必要があります。次に、ユーザーからの入力を受け付けて、データを編集したり表示したりする必要があります。私たちのプログラムは、各プロセスに対して以下のステップを実行することでこれを実現することができます。

1. ユーザーから引数を収集する。
2. アプリケーションに渡されるコマンドから、コマンド（取得、編集、削除、作成）を定義し、ToDoタイトルを定義する。
3. プログラムを過去に実行した際のToDo項目を保存するJSONファイルを読み込む。
4. プログラムに渡されたコマンドを元にget、edit、delete、createの各機能を実行し、最後にその状態の結果をJSONファイルに保存する。

この4ステップのプロセスを可能にするために、最初にserde crateで状態をロードすることから始めます。

### JSONファイルの読み出しと書き込み

現在は、JSONファイルの形でデータを永続化する段階です。第6章「PostgreSQLによるデータ永続化」で、これを適切なデータベースにアップグレードする予定です。しかし今は、Web アプリケーションの最初の依存関係である serde_json を導入するつもりです。この serde_json クレートは、Rust データから JSON データへの変換、およびその逆を処理します。次の章では、serde_json を使用して HTTP リクエストを処理する予定です。以下のコードで、Cargo.toml ファイルにこのクレートをインストールできます。

```toml
[dependencies]
serde_json="1.0.59"
```

将来的にストレージをアップグレードする予定なので、JSONファイルの読み取りと書き込みに関する操作を、アプリケーションの他の部分から分離しておくことは理にかなっています。データベースをアップグレードする際に、デバッグやリファクタリングが必要にならないようにするためです。また、JSONファイルの読み取りと書き込みの際に管理しなければならないスキーマやマイグレーションがないため、シンプルに保つことができます。これを考慮すると、必要なのは読み込みと書き込みの関数だけです。このモジュールは小さくシンプルなので、main.rsの隣に1つのファイルを置くだけでよい。まず、src/state.rsに以下のコードで必要なものをインポートする必要があります。

```rust
use std::fs::File;
use std::fs;
use std::io::Read;
use serde_json::Map;
use serde_json::value::Value;
use serde_json::json;
```

図2.5にマッピングしたように、データを読み込むためには、標準ライブラリと一連の構造体が必要です。


図2.5 JSONファイルを読み込むための手順

図2.5の手順を以下のコードで実行することができます。

```rust
pub fn read_file(file_name: &str) -> Map<String, Value> {
    let mut file = File::open(file_name.to_string()).unwrap();
    let mut data = String::new();
    file.read_to_string(&mut data).unwrap();
    let json: Value = serde_json::from_str(&data).unwrap();
    let state: Map<String, Value> = json.as_object().unwrap().clone();
    return state
}
```

先のコードでは、ファイルのオープンを直接アンラップしていることがわかります。これは、ファイルを読み取れないとプログラムを続ける意味がないためで、その結果、ファイルの読み取りを直接アンラップしています。また、この文字列をJSONデータで埋めるため、文字列はミュータブルでなければならないことにも注意しなければなりません。さらに、serde_json クレートを使って JSON データを処理し、マップに構成します。これで、プログラムの残りの部分で、このマップ変数を介してToDoアイテムにアクセスできるようになりました。次に、データを書き込む必要がありますが、これは同じファイル内で以下のコードで行うことができます。

```rust
pub fn write_to_file(file_name: &str, state: &mut Map<String, Value>) {
    let new_data = json!(state);
    fs::write(
          file_name.to_string(),
          new_data.to_string()
    )
    .expect("Unable to write file");
}
```

先のコードでは、Map変数とファイルへのパスを受け取っています。次に、serde_jsonクレートのjson！マクロを使用して、Map変数をJSONに変換します。そして、JSON データを文字列に変換し、JSON ファイルに書き込んでいます。このように、ToDo 項目を JSON ファイルに読み書きする関数が用意されました。これで、main.rsファイルをアップグレードして、JSONファイルにToDoアイテムを読み書きするシンプルなコマンドラインToDoアプリケーションを構築することができます。次のコードで、プログラムに渡される基本的な引数を使用して、このアプリケーションと対話することができます。

```rust
mod state;

use std::env;
use state::{write_to_file, read_file};
use serde_json::value::Value;
use serde_json::{Map, json};

fn main() {
    let args: Vec<String> = env::args().collect();
    let status: &String = &args[1];
    let title: &String = &args[2];
    let mut state: Map<String, Value> =
    read_file("./state.json");
    println!("Before operation: {:?}", state);
    state.insert(title.to_string(), json!(status));
    println!("After operation: {:?}", state);
    write_to_file("./state.json", &mut state);
}
```

先のコードでは、次のようにしています。

1. プログラムに渡された引数から、ステータスやタイトルを収集する
2. JSONファイルからToDo項目を読み込む
3. JSONファイルからToDo項目をプリントアウトしたもの
4. 新しいToDo項目を挿入する
5. 新しいToDoのセットをメモリからプリントアウトしたもの
6. 新しいToDoリストをJSONファイルに書き込む。

ルートパスはCargo.tomlファイルのある場所になるので、Cargo.tomlファイルの隣にstate.jsonという空のJSONファイルを定義します。このファイルを操作するには、次のコマンドを渡します。

```bash
cargo run pending washing
```

先のコマンドを実行すると、次のようなプリントアウトになります。

```
Before operation: {}
After operation: {"washing": String("pending")}
```

先の出力では、washingが挿入されたことがわかります。また、JSONファイルを確認すると、washingがファイルに書き込まれていることがわかります。to_doモジュールについて、構築したすべての構造体やtraitを含めて、一切の言及を削除したことにお気づきでしょうか。これは、忘れてしまったわけではありません。これは、to_doをstateモジュールと融合させる前に、JSONファイルとのやりとりがうまくいくかどうかをテストしているに過ぎないのです。次のセクションで、To-do構造体に実装されているtraitを修正することで、To_doとstateモジュールを融合させることにします。

### トレイトの再確認

ToDoアイテムの状態をJSONファイルで管理するモジュールを定義したので、trait関数がどのようにデータを処理し、JSONファイルと対話するかについて考えてみます。まず始めに、最も単純な形質であるsrc/to_do/traits/get.rsファイルのGet形質を更新します。ここでは、単にJSONマップからToDoアイテムを取得して、それをプリントアウトしています。JSONマップをget関数に渡して、マップからToDo項目のステータスを取得し、stateからToDo項目のタイトルを取得して、次のコードでコンソールに出力することで実現できます。

```rust
use serde_json::Map;
use serde_json::value::Value;
pub trait Get {
    fn get(&self, title: &String, state: &Map<String, Value>) {
        let item: Option<&Value> = state.get(title);
        match item {
            Some(result) => {
                println!("\n\nItem: {}", title);
                println!("Status: {}\n\n", result);
            },
            None => println!("item: {} was not found", title)
        }
    }
}
```

先のコードでは、JSON Mapに対してget関数を実行し、その結果をマッチングして、抽出したものをプリントアウトしていることがわかります。つまり、Getを実装したToDoアイテムであれば、状態からToDoアイテムを抽出してプリントアウトすることができるのです。  

次に、src/to_do/traits/create.rs ファイルにある Create trait を使って、次のステップの複雑さに進みます。これは、新しいToDo項目を挿入して状態を編集し、更新された状態をJSONに書き込むので、Get traitよりも少し複雑です。これらの手順は、次のコードで実行できます。

```rust
use serde_json::Map;
use serde_json::value::Value;
use serde_json::json;
use crate::state::write_to_file;
pub trait Create {
    fn create(
            &self,
            title: &String,
            status: &String,
            state: &mut Map<String, Value>) {
        state.insert(title.to_string(), json!(status));
        write_to_file("./state.json", state);
        println!("\n\n{} is being created\n\n", title);
    }
}
```

先のコードでは、stateモジュールのwrite_to_file関数を使って、JSONファイルに状態を保存していることがわかります。このcreate関数は、ToDo項目を削除するときに必要なテンプレートとして使うことができます。削除は、基本的にcreate関数で行ったことの逆を行うことになります。src/to_do/traits/delete.rsにあるDelete特性のdelete関数を書いてみてから、次に進むこともできます。しかし、次のコードと同じように実行されるはずです。

```rust
use serde_json::Map;
use serde_json::value::Value;
use crate::state::write_to_file;
pub trait Delete {
    fn delete(&self, title: &String,
        state: &mut Map<String, Value>) {
        state.remove(title);
        write_to_file("./state.json", state);
        println!("\n\n{} is being deleted\n\n", title);
    }
}
```

先のコードでは、JSON Mapでremove関数を使用し、更新された状態をJSONファイルに書き込んでいるに過ぎません。このセクションは終わりに近づいています。あとは、src/to_do/traits/edit.rsファイルに、Edit特性のedit関数を作成するだけです。2つの関数があります。1つはToDoアイテムのステータスをDONEに設定します。もう1つの関数は、ToDoアイテムのステータスをPENDINGに設定するものです。これらは、状態を更新し、更新された状態をJSONファイルに書き込むことで実現されます。この先を読む前に、自分で書いてみるのもいいでしょう。うまくいけば、あなたのコードは次のようなコードになるでしょう。

```rust
use serde_json::Map;
use serde_json::value::Value;
use serde_json::json;
use crate::state::write_to_file;
use super::super::enums::TaskStatus;
pub trait Edit {
    fn set_to_done(&self, title: &String,
        state: &mut Map<String, Value>) {
        state.insert(title.to_string(),
        json!(TaskStatus::DONE.stringify()));
        write_to_file("./state.json", state);
        println!("\n\n{} is being set to done\n\n", title);
    }
    fn set_to_pending(&self, title: &String,
        state: &mut Map<String, Value>) {
        state.insert(title.to_string(),
        json!(TaskStatus::PENDING.stringify()));
        write_to_file("./state.json", state);
        println!("\n\n{} is being set to pending\n\n", title);
    }
}
```

これで、JSONファイルを介さずに、最初に望んだ処理を実行することができるようになりました。これらの特性を最初に定義したときのように、main.rsファイルでこれらの特性を直接利用することを妨げるものはありません。しかし、これでは拡張性がありません。というのも、私たちは基本的に、複数のビューとAPIエンドポイントを持つWebアプリケーションを構築することになります。そのため、複数の異なるファイルで、これらの特性やストレージ処理とやり取りすることになります。したがって、コードを繰り返すことなく、標準化された方法でこれらの形質と対話する方法を考え出す必要がありそうです。

### トレイトと構造体の処理

私たちのコードがシンプルなインターフェイスと対話し、最小限の痛みで更新できるようにし、繰り返しのコードとその結果生じるエラーを減らすために、図2.6に見られるように、プロセス層が必要です。


図2.6 - プロセスモジュールを介した形質転換の流れ

図2.6では、構造体が緩やかな方法でtraitに結合されており、JSONとの間のデータフローがtraitを経由していることがわかります。また、processsモジュールに1つのエントリポイントがあり、そのエントリポイントがコマンドを適切なtraitに導き、そのtraitが必要なときにJSONファイルにアクセスすることもわかります。traitの定義ができたので、あとはprocesssモジュールを構築してtraitに接続し、main.rsファイルに接続するだけです。processsモジュール全体をsrc/processes.rsファイル1つで構築することになります。1つのファイルにしているのは、データベースを扱うときに削除するためです。将来的に削除することが分かっているのであれば、あまり大きな技術的負債を背負う必要はないでしょう。とりあえず、以下のコードで必要な構造体とtraitをすべてインポートして、processsモジュールの構築を開始します。

```rust
use serde_json::Map;
use serde_json::value::Value;
use super::to_do::ItemTypes;
use super::to_do::structs::done::Done;
use super::to_do::structs::pending::Pending;
use super::to_do::traits::get::Get;
use super::to_do::traits::create::Create;
use super::to_do::traits::delete::Delete;
use super::to_do::traits::edit::Edit;
```

これで、非公開の関数を構築し始めることができます。まず、Pending構造体を処理することから始めましょう。次のコードにあるように、保留中のToDoアイテムを取得、作成、編集できることが分かっています。

```rust
fn process_pending(
        item: Pending,
        command: String, 
        state: &Map<String, Value>) {
    let mut state = state.clone();
    match command.as_str() {
        "get" => item.get(&item.super_struct.title, &state),
        "create" => item.create(
                &item.super_struct.title, 
                &item.super_struct.status.stringify(),
                &mut state),
        "edit" => item.set_to_done(
                &item.super_struct.title, 
                &mut state),
        _ => println!("command: {} not supported", command)
    }
}
```

先のコードでは、Pending構造体、コマンド、ToDoアイテムの現在の状態を取り込んでいることがわかります。次に、コマンドをマッチングし、そのコマンドに関連するtraitを実行します。渡されたコマンドがget、create、editのいずれでもない場合は、それをサポートせず、サポートされていないコマンドをユーザーに伝えるエラーを投げます。これはスケーラブルです。たとえば、Pending構造体がJSONファイルからToDo項目を削除できるようにする場合、Pending構造体にDelete traitを実装して、process_pending関数にdeleteコマンドを追加するだけでよい。これは合計で2行のコードで済み、この変更はアプリケーション全体に適用されます。この変更は、コマンドを削除した場合にも適用されます。これで、Pending構造体を柔軟に実装できるようになりました。このことを念頭に置いて、この先を読む前にprocess_done関数をコーディングすることを選択することができます。もしそうするのであれば、次のようなコードになることを期待します。

```rust
fn process_done(
        item: Done,
        command: String, 
        state: &Map<String, Value>) {
    let mut state = state.clone();
    match command.as_str() {
        "get"    => item.get(&item.super_struct.title, &state),
        "delete" => item.delete(&item.super_struct.title, &mut state),
        "edit"   => item.set_to_pending(&item.super_struct.title, &mut state),
        _        => println!("command: {} not supported", command)
    }
}
```

これで、両方の構造体を処理できるようになりました。ここで、モジュールを設計する際に構造体のスケーラビリティが活きてきます。コマンドと同様に、構造体もtraitと同じようにスタックするようにします。そこで、図2.7に示すように、エントリーポイントを作成します。


図2.7 プロセスのスケーラビリティ

図2.7から、エントリポイントによる経路を増やすことで、構造体へのアクセスを拡張できることがわかります。このことをより理解するために、次のようなコードでエントリポイント（今回はpublic関数）を定義しておきましょう。

```rust
pub fn process_input(item: ItemTypes, command: String, state: &Map<String, Value>) {
    match item {
        ItemTypes::Pending(item) => process_pending(item, command, state),
        ItemTypes::Done(item) => process_done(item, command, state)
    }
}
```

先のコードでは、ItemTypes列挙型で正しい構造体にルートしていることがわかります。このモジュールでは、新しい構造体をItemTypes列挙型に追加し、processsモジュールでその構造体を処理する新しい関数を記述し、構造体に必要な特性を適用することによって、さらに多くの構造体を処理できます。これでprocesssモジュールは完全に完成したので、これを利用するためにmain.rsファイルを書き換えることができます。まず、以下のコードで必要なものをインポートします。

```rust
mod state;
mod to_do;
mod processes;

use std::env;
use serde_json::value::Value;
use serde_json::Map;
use state::read_file;
use to_do::to_do_factory;
use to_do::enums::TaskStatus;
use processes::process_input;
```

これらのインポートにより、JSONファイルからデータを読み取り、to_do_factoryを使用して環境から収集した入力から構造体を作成し、これをprocesssモジュールに渡してJSONファイルを更新していることがわかります。この時点で、読むのをやめて、自分でこの処理をコード化してみるのがよいでしょう。JSONファイルからデータを取得し、ToDo項目のタイトルがすでにJSONファイルに格納されているかどうかをチェックする必要があることを忘れないでください。JSONファイルのデータからタイトルを見つけることができなければ、完了したタスクを作成することができないので、保留状態になることがわかります。この方法を選択した場合、あなたのコードはうまくいけば次のようになります。

```rust
fn main() {
    let args: Vec<String> = env::args().collect();
    let command: &String = &args[1];
    let title: &String = &args[2];
    let state: Map<String, Value> = read_file("./state.json");
    let status: String;
    match &state.get(*&title) {
        Some(result) => {
            status = result.to_string().replace('\"', "");
        }
        None => {
            status = "pending".to_owned();
        }
    }
    let item = to_do_factory(title, 
                             TaskStatus::from_string(
                             status.to_uppercase()));
    process_input(item, command.to_string(), &state);
}
```

何かを実行する前に、from_string関数を使用してTaskStatusのインスタンスを作成したことにお気づきかもしれませんね。まだ、from_string関数を構築していません。この時点で、impl TaskStatusブロックの中で、自分で構築できるようになるはずです。from_string関数を構築しようとすると、src/to_do/enums.rsファイルにある以下のようなコードになるはずです。

```rust
impl TaskStatus {
    . . .
    pub fn from_string(input_string: String) -> Self {
        match input_string.as_str() {
            "DONE" => TaskStatus::DONE,
            "PENDING" => TaskStatus::PENDING,
            _ => panic!("input {} not supported", input_string)
        }
    }
}
```

もし、私たちが作成したインターフェイスを利用して、プログラムを実行できたのなら、よくやったと言えるでしょう。これで、メイン関数で様々な処理を簡単にオーケストレーションできるようになったことがわかります。次のコマンドで、このプログラムと対話することができます。

```bash
cargo run create washing
```

先のコマンドは、JSONファイルに「洗濯」というToDo項目を作成し、ステータスを「保留」としています。このほかにも、コマンドラインから実行できる機能はすべてサポートされています。これで、JSONファイルにToDo項目を保存する基本的なコマンド・アプリケーションを構築しました。しかし、これは単なる基本的なコマンドラインアプリケーションではありません。私たちはモジュールを構成することで、拡張性と柔軟性を持たせています。  
