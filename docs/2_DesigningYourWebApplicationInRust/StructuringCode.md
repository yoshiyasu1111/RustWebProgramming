## コードの構造化

これで、Webアプリケーションを構築する旅に出ることができます。この章の残りの部分では、Webフレームワークに触れたり、HTTPリスナーを構築したりすることはありません。これは次章で行います。しかし、JSONファイルと対話するToDoモジュールを構築します。このモジュールは、最小限の労力で構築したWebアプリケーションに挿入できるような構造になっています。このToDoモジュールでは、ToDoアイテムの作成、更新、削除を行うことができます。そして、コマンドラインを使ってこれを操作することになります。ここでのプロセスは、拡張性と柔軟性を備えた、構造化されたコードを構築する方法を探ることです。このことを理解するために、このモジュールの構築を次のようなチャンクに分解していきます。

1. 保留中のToDo項目と完了したToDo項目に対する構造体を構築します。
2. 構造体をモジュール内で最小限のクリーンな入力で構築できるようにするファクトリーを構築する。
3. 構造体がToDo項目を削除、作成、編集、取得できるようにするためのtraitを構築する。
4. ToDoアイテムを保存するための読み書き可能なファイルモジュールを構築する（後の章で適切なデータベースに置き換える予定です）。
5. 設定ファイル内の変数に基づいてアプリケーションの動作を変更できるconfigモジュールを構築する。

これらのステップに取り掛かる前に、アプリケーションを実行する必要があります。アプリケーションを格納したいディレクトリに移動して、todo_appという新しいCargoプロジェクトを開始します。この後、to_doモジュールにToDoアイテムの管理を行うロジックを配置します。これは、以下のレイアウトのように、to_doディレクトリを作成し、そのディレクトリのベースにmod.rsファイルを置くことで実現できます。

```
├── main.rs
└── to_do
    ├── mod.rs
```

この構造体を使って、to_doモジュールを構造体から構築していきます。To_doファイルについては、最初のステップで、完了したTo_do項目と保留中のTo_do項目の構造体を構築しているので、今は気にしないでください。

### ToDo構造体の構築

現在、ToDoの構造体としては、実行待ちのものと実行済みのものの2つしかありません。しかし、他のカテゴリを導入することもできます。たとえば、バックログカテゴリを追加したり、開始されたものの何らかの理由でブロックされているタスクのための保留タスクを追加したりすることができます。間違いやコードの繰り返しを避けるために、Base構造体を作成し、その構造体を他の構造体で利用できるようにすることができます。Base構造体には、共通のフィールドと関数が格納されています。Base構造体を変更すると、他のすべてのToDo構造体にも伝搬します。また、ToDoアイテムの種類を定義する必要があります。pendingとdoneに文字列をハードコードすることもできますが、これは拡張性がなく、エラーが発生しやすいという問題があります。これを避けるために、enumを使ってToDoアイテムの種類を分類し、プレゼンテーションを定義することにします。これを実現するために、次のようなファイル構造をモジュールに作成する必要があります。

```
├── main.rs
└── to_do
    ├── enums.rs
    ├── mod.rs
    └── structs
        ├── base.rs
        ├── done.rs
        ├── mod.rs
        └── pending.rs
```

先のコードでは、2つのmod.rsファイルがあることに気づきます。これらのファイルは、基本的にファイルを宣言する場所であり、同じディレクトリ内の他のファイルからアクセスできるようにするために、その中で定義するものです。また、mod.rsファイルでファイルを公開すれば、ディレクトリの外からアクセスできるようにすることもできます。コードを書く前に、図2.3で、モジュール内のデータの流れを確認しましょう。  


図2.3-ToDoモジュールにおけるデータの流れ

Base構造体は、他のToDo構造体から利用されていることがわかります。Base構造体を宣言しなければ、他のToDo構造体はBase構造体にアクセスすることができません。しかし、to_do/structsディレクトリの外にはBase構造体を参照するファイルはないので、public宣言である必要はありません。

さて、このモジュールのデータフローを理解したところで、図2.3を振り返って、最初に取り組むべきことを整理する必要があります。この列挙型には依存関係がないことがわかります。実際、このenumはすべての構造体を供給しています。したがって、/to_do/enums.rsファイルにあるenumから始めることにします。このenumは、次のコードでタスクのステータスを定義しています。

```rust
pub enum TaskStatus {
    DONE,
    PENDING
}
```

これは、タスクのステータスを定義することになると、コード上で機能します。しかし、ファイルやデータベースに書き込む場合は、列挙型を文字列で表現できるようにするメソッドを構築する必要があります。そのために、次のようなコードでTaskStatus列挙型のstringify関数を実装します。  

```rust
impl TaskStatus {
    pub fn stringify(&self) -> String {
        match &self {
            &Self::DONE => {"DONE".to_string()},
            &Self::PENDING => {"PENDING".to_string()}
        }
    }
}
```

これを呼び出すと、ToDoタスクのステータスをコンソールに出力し、JSONファイルに書き込むことができる。

備考

stringify関数は動作しますが、enumの値を文字列に変換する別の方法があります。文字列変換を実現するためには、TaskStatusのDisplay traitを実装すればよい。まず、以下のコードでformatモジュールをインポートする必要があります。

```rust
use std::fmt;
```

そして、以下のコードでTaskStatus structのDisplay traitを実装することができます。

```rust
impl fmt::Display for TaskStatus {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match &self {
            &Self::DONE => {write!(f, "DONE")},
            &Self::PENDING => {write!(f, "PENDING")}
        }
    }
}
```

このtraitの実装は、私たちのstringify関数と同じロジックを持っています。しかし、このtraitは必要なときに利用されます。つまり、次のようなコードになります。

```rust
println!("{}", TaskStatus::DONE);
println!("{}", TaskStatus::PENDING);
let outcome = TaskStatus::DONE.to_string();
println!("{}", outcome);
```

これにより、以下のように印刷されます。

```
DONE
PENDING
DONE
```

ここでは、TaskStatusをprintln！に渡すと、自動的にDisplay traitが利用されることがわかります。

次のコードで、/to_do/mod.rsファイル内でenumを公開できるようにしました。

```rust
pub mod enums;
```

ここで、図2.3を参照して、次に構築できるものを確認します。それはBase構造体です。Base構造体は、/to_do/structs/base.rsファイルに以下のコードで定義することができます。

```rust
use super::super::enums::TaskStatus;
pub struct Base {
    pub title: String,
    pub status: TaskStatus
}
```

ファイル先頭のimportから、super::superを使ってTaskStatus enumにアクセスすることができます。TaskStatus列挙型は上位のディレクトリにあることがわかります。このことから、superを使えば、カレントディレクトリのmod.rsファイルで宣言されているものにアクセスできることが推測できる。つまり、/to_do/structs/ディレクトリにあるファイルでsuper::superを使用すると、/to_do/mod.rsファイルで定義されているものにアクセスできるようになります。

これで、/to_do/structs/mod.rsファイルにおいて、以下のコードでBase structを宣言することができました。

```rust
mod base;
```

Base構造体は/to_do/structs/ディレクトリの外ではアクセスされないので、publicとして宣言する必要はありません。さて、図2.3を振り返ると、Pending構造体とDone構造体を構築していることがわかります。このとき、/to_do/structs/pending.rsファイルでは、次のようなコードでBase構造体を利用するコンポジションを使用しています。

```rust
use super::base::Base;
use super::super::enums::TaskStatus;
pub struct Pending {
    pub super_struct: Base
}
impl Pending {
    pub fn new(input_title: &str) -> Self {
        let base = Base {
            title: input_title.to_string(),
            status: TaskStatus::PENDING
        };
        return Pending{super_struct: base}
    }
}
```

先ほどのコードで、super_structフィールドにBase構造体が格納されていることが確認できました。ここでは、enumを利用して、ステータスをpendingと定義しています。つまり、コンストラクタにタイトルを渡すだけで、タイトルとステータスがpendingの構造体ができあがります。このように考えると、Done構造体のコーディングは、/to_do/structs/done.rsファイルの次のコードで簡単にできるはずです。

```rust
use super::base::Base;
use super::super::enums::TaskStatus;
pub struct Done {
    pub super_struct: Base
}
impl Done {
    pub fn new(input_title: &str) -> Self {
        let base = Base {
            title: input_title.to_string(),
            status: TaskStatus::DONE
        };
        return Done{super_struct: base}
    }
}
```

TaskStatus列挙型にDONEステータスがあることを除けば、Pending構造体の定義と大きな違いはないことがわかります。次のコードで、/to_do/structs/mod.rsファイルのディレクトリの外でも構造体を利用できるようにしました。

```rust
mod base;
pub mod done;
pub mod pending;
```

次に、/to_do/mod.rsファイル内で以下のコードで構造体を宣言することで、main.rsファイル内で構造体にアクセスできるようになります。

```rust
pub mod structs;
pub mod enums;
```

これで、基本的なモジュールを作成し、main.rs ファイルに公開しました。とりあえず、このモジュールを使って、保留中のタスクと完了したタスクを作成する基本的なコードを書きます。これは、次のようなコードで実現できます。

```rust
mod to_do;
use to_do::structs::done::Done;
use to_do::structs::pending::Pending;
fn main() {
    let done = Done::new("shopping");
    println!("{}", done.super_struct.title);
    println!("{}", done.super_struct.status.stringify());
    let pending = Pending::new("laundry");
    println!("{}", pending.super_struct.title);
    println!("{}", pending.super_struct.status.stringify()
    );
}
```

先のコードでは、to_doモジュールを宣言したことがわかります。次に、構造体をインポートして、pending構造体とdone構造体を作成しました。このコードを実行すると、次のようなプリントアウトが表示されます。

```
shopping
DONE
laundry
PENDING
```

これにより、main.rsファイルが過剰なコードで負荷がかかるのを防ぐことができます。もし、保留アイテムやバックログアイテムなど、作成できるアイテムの種類を増やそうとすると、main.rsのコードは膨れ上がってしまうでしょう。そこで、次のステップで説明するファクトリーの出番です。

### ファクトリーで構造体を管理する

ファクトリーパターンは、構造体の構築をモジュールのエントリポイントに抽象化するものです。私たちのモジュールでは、次のように動作させることができます。


図2.4 - ToDoファクトリーの流れ

ファクトリーが行うことは、インターフェースを提供することでモジュールを抽象化することです。私たちはモジュールの構築を楽しんでいますが、他の開発者がこのモジュールを使いたいと思ったとき、シンプルなファクトリーインターフェイスと優れたドキュメントがあれば、開発者の時間を大幅に節約することができます。開発者がすべきことは、いくつかのパラメータを渡して、構築された構造体をenumに包んでファクトリーから取り出すことだけです。もし、モジュールの内部を変更したり、より複雑になったとしても、これは問題にはならないでしょう。他のモジュールがこのインターフェイスを使用していても、インターフェイスの一貫性を保てば、変更によって他のコードが壊れることはない。ファクトリーを構築するには、/to_do/mod.rsファイルに次のようなコードでファクトリー関数を定義します。

```rust
pub mod structs;
pub mod enums;

use enums::TaskStatus;
use structs::done::Done;
use structs::pending::Pending;

pub enum ItemTypes {
    Pending(Pending),
    Done(Done)
}

pub fn to_do_factory(title: &str, status: TaskStatus) -> ItemTypes {
    match status {
        TaskStatus::DONE => {
            ItemTypes::Done(Done::new(title))
        },
        TaskStatus::PENDING => {
            ItemTypes::Pending(Pending::new(title))
        }
    }
}
```

前のコードでは、ItemTypesというenumを定義し、構築されたタスク構造体をパッケージ化しています。ファクトリー関数は、基本的に入力されたタイトルとステータスを受け取ります。そして、ファクトリーは入力されたステータスにマッチします。入力されたステータスがどのようなものかを確認したら、そのステータスにマッチしたタスクを構築し、ItemTypes enumで囲みます。これは、どんどん複雑になっていきますが、メインファイルには関係ありません。このファクトリーをmain.rsファイルに実装すると、次のようなコードになります。

```rust
mod to_do;

use to_do::to_do_factory;
use to_do::enums::TaskStatus;
use to_do::ItemTypes;

fn main() {
    let to_do_item = to_do_factory("washing", TaskStatus::DONE);
    match to_do_item {
        ItemTypes::Done(item) => {
            println!("{}", item.super_struct.status.stringify());
            println!("{}", item.super_struct.title);
        },
        ItemTypes::Pending(item) => {
            println!("{}", item.super_struct.status.stringify());
            println!("{}", item.super_struct.title);
        }
    }
}
```

先のコードでは、ToDo項目に対して作成したいパラメータをファクトリーに渡し、その結果をマッチングして項目の属性をプリントしていることがわかります。main.rsファイルには、さらに多くのコードが導入されています。これは、返されたenumのラップを解除して属性を出力することで、デモを行うためです。通常、このラップされたenumを他のモジュールに渡して処理させることになります。構造体を作成するには、次のような1行のコードが必要なだけです。

```rust
let to_do_item = to_do_factory("washing", TaskStatus::DONE);
```

つまり、ToDo項目を作成し、enumに包まれているため、ほとんど手間をかけずに受け渡すことができるのです。他の関数やモジュールは、enumを受け入れればよいのです。このように、柔軟性があることがわかります。しかし、このコードでは、Base構造体、Done構造体、Pending構造体を廃止して、ステータスとタイトルを受け取る1つの構造体だけにすることができます。そうすれば、コードが少なくなります。しかし、柔軟性は失われます。次のステップでは、構造体にtraitを追加して機能をロックダウンし、安全性を確保します。

### トレイトによる機能定義

現在、この構造体は、タスクのステータスやタイトルを保持する以外には何もしていません。しかし、構造体にはさまざまな機能を持たせることができます。そのため、わざわざ個別の構造体を定義しています。例えば、ここではToDoアプリケーションを構築しています。アプリケーションをどのように構成するかは最終的にあなた次第ですが、完了したタスクを作成できないようにするのは無理なことではありません。この例は些細なことに思えるかもしれません。本書では、解決する問題をシンプルにするためにToDoリストを使っています。このため、解決する問題の理解に時間をかけずに、RustでWebアプリケーションを開発するための技術的な側面に集中することができます。しかし、銀行取引を処理するシステムのような、より要求の厳しいアプリケーションでは、ロジックの実装方法を厳密にし、望ましくない処理が起こる可能性をロックする必要があることを認識する必要があります。ToDoアプリケーションでは、各プロセスに個別のtraitを作成し、それを必要なタスク構造体に割り当てることでこれを実現することができます。そのためには、to_doモジュール内にtraitディレクトリを作成し、それぞれのtraitに対応するファイルを作成する必要があります（以下のような構成になります）。

```
├── mod.rs
└── traits
    ├── create.rs
    ├── delete.rs
    ├── edit.rs
    ├── get.rs
    └── mod.rs
```

そして、以下のコードでto_do/traits/mod.rsファイル内のすべての特性を公開定義することができます。

```rust
pub mod create;
pub mod delete;
pub mod edit;
pub mod get;
```

また、to_do/mod.rsファイルにおいて、以下のコードで特徴を公的に定義する必要があります。

```rust
pub mod traits;
```

これで、モジュール内にすべての trait ファイルが配置されたので、trait の構築を開始することができます。まず、to_do/traits/get.rs ファイルに以下のコードで Get trait を定義します。

```rust
pub trait Get {
    fn get(&self, title: &str) {
        println!("{} is being fetched", title);
    }
}
```

これは単にtraitの適用方法を示すものなので、とりあえず何が起こっているかをプリントアウトするだけにします。ただし、これを実装しているtraitのget関数を上書きすることは可能です。Editというtraitに関しては、to_do/traits/edit.rsファイルに以下のコードで状態を変更する関数を2つ用意することができます。

```rust
pub trait Edit {
    fn set_to_done(&self, title: &str) {
        println!("{} is being set to done", title);
    }
    fn set_to_pending(&self, title: &str) {
        println!("{} is being set to pending", title);
    }
}
```

ここでパターンを見ることができます。そこで、念のため、Create 特性は to_do/traits/create.rs ファイルで次のような形式をとります。

```rust
pub trait Create {
    fn create(&self, title: &str) {
        println!("{} is being created", title);
    }
}
```

Delete特性は、to_do/traits/delete.rsファイルに以下のコードで定義されています。

```rust
pub trait Delete {
    fn delete(&self, title: &str) {
        println!("{} is being deleted", title);
    }
}
```

これで、必要な特性はすべて定義できました。これを利用して、ToDoアイテム構造体の動作を定義し、ロックダウンすることができます。Done構造体では、以下のコードでto_do/structs/done.rsファイルにtraitをインポートすることができます。

```rust
use super::super::traits::get::Get;
use super::super::traits::delete::Delete;
use super::super::traits::edit::Edit;
```

そして、Done structの定義後の同じファイル内で、以下のコードでDone structを実装することができます。

```rust
impl Get for Done {}
impl Delete for Done {}
impl Edit for Done {}
```

これで、Done構造体は、ToDo項目を取得、編集、削除できるようになりました。ここで、第1章「Rust入門」で強調したように、traitの威力を実感することができます。traitは、簡単に積み重ねたり削除したりすることができます。例えば、完了したToDo項目を作成できるようにするには、Create for Done;という単純なインプリメントで実現できます。Done構造体に必要なtraitを定義できたので、次にPending構造体を作成し、to_do/structs/pending.rsファイルに必要なものをインポートして、次のコードを記述します。

```rust
use super::super::traits::get::Get;
use super::super::traits::edit::Edit;
use super::super::traits::create::Create;
```

そして、Pending structの定義の後に、以下のコードでこれらの特徴を実装することができます。

```rust
impl Get for Pending {}
impl Edit for Pending {}
impl Create for Pending {}
```

前述のコードでは、Pending構造体は取得、編集、作成が可能ですが、削除はできないことがわかります。また、これらの特性を実装することで、Pending構造体とDone構造体をコンパイルすることなく結びつけることができます。たとえば、Edit特性を実装した構造体を受け入れた場合、その構造体はPending構造体とDone構造体の両方を受け入れることになります。しかし、Delete特性を実装した構造体を受け入れる関数を作成すると、Done構造体は受け入れられますが、Pending構造体は拒否されます。このように、積極的な型チェックと柔軟性が見事に調和しており、まさにRustの設計の真価が発揮されています。これで、構造体に必要な特性がすべて揃ったので、これを利用してmain.rsファイルを完全に書き換えることができます。まず、以下のコードで必要なものをインポートします。

```rust
mod to_do;
use to_do::to_do_factory;
use to_do::enums::TaskStatus;
use to_do::ItemTypes;
use crate::to_do::traits::get::Get;
use crate::to_do::traits::delete::Delete;
use crate::to_do::traits::edit::Edit;
```

今回注意すべきはインポートです。せっかく構造体にtraitを実装したのに、それを使用するファイルにはtraitをインポートしなければなりません。これは少し分かりにくいかもしれません。例えば、構造体を初期化した後にGet traitからget関数を呼び出すと、item.get(&item.super_struct.title);のような形になる。get関数は、初期化された構造体に紐づいているのです。直感的には、traitをインポートする必要がないのは理にかなっています。しかし、traitをインポートしないと、コンパイラやIDEから「getという名前の関数が構造体の中に見つからない」という不親切なエラーが出ます。これは、将来的にデータベースクレートやWebフレームワークのtraitを使用するようになり、パッケージ構造体を使用するためにこれらのtraitをインポートする必要があるため重要です。インポートが完了したら、次のコードでmain関数の中でtraitとfactoryを利用することができます。

```rust
fn main() {
    let to_do_items = to_do_factory("washing", TaskStatus::DONE);
    match to_do_items {
        ItemTypes::Done(item) => {
            item.get(&item.super_struct.title);
            item.delete(&item.super_struct.title);
        },
        ItemTypes::Pending(item) => {
            item.get(&item.super_struct.title);
            item.set_to_done(&item.super_struct.title);
        }
    }
}
```

先のコードを実行すると、次のようなプリントアウトが得られます。

```
washing is being fetched
washing is being deleted
```

ここで行ったことは、エントリーポイントを含む独自のモジュールを構築したことです。そして、それをmain関数にインポートして実行しました。さて、基本的な構造は構築され、動作していますが、このモジュールが役に立つように、変数を渡したり、ファイルに書き込んだりして、環境と相互作用するようにする必要があります。
