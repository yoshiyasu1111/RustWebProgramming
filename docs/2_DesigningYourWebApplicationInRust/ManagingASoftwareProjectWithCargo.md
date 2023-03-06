## Cargoによるソフトウェアプロジェクトの管理

Cargoを使ってプログラムの構造化を始める前に、基本的な単一ファイルのアプリケーションを構築する必要があります。そのためには、まずローカルディレクトリにhello_world.rsというファイルを作成する必要があります。拡張子.rsは、そのファイルがRustファイルであることを表しています。正直なところ、拡張子が何であるかは重要ではありません。そのファイルに実行可能なRustコードが書かれていれば、コンパイラは何の問題もなくコンパイルして実行します。しかし、拡張子が異なると、他の開発者やコード編集者を混乱させたり、他のRustファイルからコードをインポートする際に問題が発生する可能性があります。そのため、Rustファイルの命名には.rsを使用するのがベストです。hello_world.rsファイルの内部には、次のようなコードがあります。

```rust
fn main() {
    println!("hello world");
}
```

これは、前章の最初のコードブロックと何ら変わりはありません。さて、hello_world.rsファイルにエントリポイントを定義したので、次のコマンドでファイルをコンパイルすることができます。

```bash
rustc hello_world.rs
```

コンパイルが終了すると、同じディレクトリに実行可能なバイナリファイルが作成されます。Windows上でコンパイルした場合、以下のコマンドでバイナリを実行することができます。

```bash
.\hello_world.exe
```

LinuxやmacOSでコンパイルした場合、以下のコマンドで実行できます。

```bash
./hello_world
```

単純なhello worldの例しか作っていないので、hello worldはただプリントアウトされるだけです。この方法は、1つのファイルで簡単なアプリケーションを構築する場合には便利ですが、複数のファイルにまたがるプログラムを管理する場合には推奨されません。サードパーティのモジュールに依存する場合にもお勧めできません。そこで、Cargoの出番です。Cargoは、実行、テスト、ドキュメント作成、ビルド/コンパイル、サードパーティ製モジュールの依存関係など、すべてをいくつかの簡単なコマンドですぐに管理します。この章では、これらのコマンドについて説明します。hello worldの例を実行したときに見たところ、コードを実行する前にコンパイルする必要があります。そこで、次のセクションに進み、Cargoを使用して基本的なアプリケーションを構築してみましょう。


### Cargoを使ったビルド

Cargoを使ったビルドは簡単です。プロジェクトをビルドしたいディレクトリに移動して、以下のコマンドを実行するだけです。

```bash
cargo new web_app
```

先のコマンドで、基本的なCargo Rustプロジェクトが構築されます。このアプリケーションを探索すると、次のような構造になっていることがわかります。

```
└── web_app
    ├── Cargo.toml
    └── src
         └── main.rs
```

Rustファイルは1つだけで、これはsrcディレクトリに収められているmain.rsファイルであることがわかります。main.rsファイルを開くと、これは前節で作ったファイルと同じであることがわかります。これは、デフォルトのコードがコンソールにhello worldをプリントアウトするエントリーポイントです。プロジェクトの依存関係やメタデータは、Cargo.tomlファイルに定義されています。プログラムを実行したい場合は、main.rsファイルに移動してrustcを実行する必要はありません。 代わりに、Cargoを使用して、以下のコマンドで実行することができます。

```bash
cargo run
```

このようにすると、プロジェクトがコンパイルされて実行され、次のようなプリントアウトが表示されます。

```
  Compiling web_app v0.1.0 (/Users/maxwellflitton/Documents/
   github/books/Rust-Web_Programming-two/chapter02/web_app)
    Finished dev [unoptimized + debuginfo] target(s) in 0.15s
     Running `target/debug/web_app`
hello world
```

ベースディレクトリが違うので、プリントアウトは少し違うでしょう。一番下にhello worldと表示されますが、これは私たちが期待するものです。また、プリントアウトには、コンパイルが最適化されていないこと、target/debug/web_appで実行されていることが記載されていることが確認できます。バイナリが保存されている場所なので、前のセクションで行ったのと同じように、target/debug/web_appバイナリに直接移動して実行することができます。targetディレクトリには、プログラムをコンパイルしたり、実行したり、ドキュメントを作成したりするためのファイルが格納されています。GitHubのリポジトリにコードを添付する場合は、.gitignoreファイルにターゲット・ディレクトリを記述して、GitHubに無視されるようにする必要があります。今、私たちは最適化されていないバージョンを実行しています。つまり、処理速度は遅いですが、コンパイルは速いです。これは、開発中に何度もコンパイルすることになるため、理にかなっています。しかし、最適化されたバージョンを実行したい場合は、次のコマンドを使用します。

```bash
cargo run --release
```

先のコマンドで、次のようなプリントアウトが得られました。

```bash
    Finished release [optimized] target(s) in 2.63s
     Running `target/release/web_app`
hello world
```

先の出力では、最適化されたバイナリがtarget/release/web_appのパスにあることがわかります。これで基本的なビルドは完了したので、次はCargoを使ってサードパーティのクレートを利用することができます。

### Cargoでクレートを出荷する

サードパーティライブラリはクレートと呼ばれます。Cargoでこれらを追加し、管理するのは簡単です。このセクションでは、https://rust-random.github.io/rand/rand/index.html にある rand crate を利用して、このプロセスを探ります。このクレートのドキュメントは、構造体、trait、およびモジュールへのリンクが明確でよく構成されていることに留意する必要があります。これは、randクレートそのものを反映しているわけではありません。これは、次のセクションで取り上げるRustの標準的なドキュメントです。このクレートをプロジェクトで使用するために、Cargo.tomlファイルを開き、以下のように[dependencies]セクションの下にrand crateを追加しています。

```toml
[dependencies]
rand = "0.7.3"
```

依存関係の定義ができたので、rand crateを使って乱数生成器を作ってみましょう。

```rust
use rand::prelude::*;
fn generate_float(generator: &mut ThreadRng) -> f64 {
    let placeholder: f64 = generator.gen();
    return placeholder * 10.0
}
fn main() {
    let mut rng: ThreadRng = rand::thread_rng();
    let random_number = generate_float(&mut rng);
    println!("{}", random_number);
}
```

先のコードでは、generate_floatという関数を定義し、クレートを使って0から10までのfloatを生成して返しています。この関数は、0から10までの浮動小数点を生成して返すもので、これを実行すると、その数値を表示します。randクレートの実装は、randのドキュメントで扱われています。このuse文はrand crateをインポートしています。randクレートを使ってfloatを生成する場合、ドキュメントではrand::preludeモジュールからインポート(*)するように指示されており、crateドキュメント(https://rust-random.github.io/rand/rand/prelude/index.html)にあるように、共通項目のインポートが簡略化されています。

ThreadRng structは、0から1の間のf64値を生成する乱数生成器です。rand crate documentation https://rust-random.github.io/rand/rand/rngs/struct.ThreadRng.html で詳しく解説されています。

さて、次はドキュメントの威力を見てみましょう。randドキュメントの紹介ページを数回クリックするだけで、デモで使用した構造体や関数の宣言を掘り下げることができます。これでコードがビルドされたので、cargo run コマンドでプログラムを実行することができます。Cargoがコンパイルしている間、randクレートからコードを取り出し、それをバイナリにコンパイルしています。また、cargo.lock ファイルが存在することも確認できました。cargo.tomlは依存関係を記述するためのものですが、cargo.lockはCargoによって生成され、依存関係の正確な情報を含んでいるので、自分で編集する必要はありません。このようなシームレスな機能と使いやすいドキュメントの組み合わせは、Rustが言語の品質だけでなく、開発エコシステムを介して限界まで開発プロセスを向上させることを示します。しかし、このようなドキュメンテーションによる利益は、純粋にサードパーティのライブラリに依存しているわけではなく、私たち自身のドキュメントを自動生成することも可能です。

### Cargoでドキュメントを作成する

Rustのような新しい言語を選んで開発するメリットは、スピードと安全性だけではありません。ソフトウェアエンジニアリングのコミュニティは、長年にわたって学び、成長し続けています。優れたドキュメントのような単純なことが、プロジェクトの成否を左右することもあるのです。これを実証するために、以下のコードでRustファイル内にMarkdown言語を定義することができます。

```rust
/// This function generates a float number using a number
/// generator passed into the function.
///
/// # Arguments
/// * generator (&mut ThreadRng): the random number
/// generator to generate the random number
///
/// # Returns
/// (f64): random number between 0 -> 10
fn generate_float(generator: &mut ThreadRng) -> f64 {
    let placeholder: f64 = generator.gen();
    return placeholder * 10.0
}
```

先のコードでは、Markdownを///マーカーで表記しています。これは、コードを見た他の開発者にこの関数が何をするものかを伝えることと、自動生成でMarkdownをレンダリングすることの2つを目的としています。documentコマンドを実行する前に、基本的なユーザー構造体と基本的なユーザー特性を定義して文書化し、これらがどのように文書化されるかを示すこともできます。

```rust
/// This trait defines the struct to be a user.
trait IsUser {
    /// This function proclaims that the struct is a user.
    ///
    /// # Arguments
    /// None
    ///
    /// # Returns
    /// (bool) true if user, false if not
    fn is_user() -> bool {
        return true
    }
}
/// This struct defines a user
///
/// # Attributes
/// * name (String): the name of the user
/// * age (i8): the age of the user
struct User {
    name: String,
    age: i8
}
```

これで、さまざまな構造をドキュメント化できたので、次のコマンドで自動ドキュメント化処理を実行することができます。

```bash
cargo doc --open
```

ドキュメントがrand crateと同じようにレンダリングされていることがわかります。


図2.1 - ウェブアプリのドキュメントビュー

先のスクリーンショットでは、web_appがクレートであることがわかります。また、randクレートのドキュメントが関与していることもわかります（スクリーンショットの左下を見ると、web_appクレートのドキュメントのすぐ上にrandクレートのドキュメントが表示されています）。User構造体をクリックすると、次の図のように、構造体の宣言、属性のために書いたMarkdown、およびtraitの意味合いが表示されます。


図2.2 - structのドキュメント

なお、本書の今後のセクションでは、可読性を維持するために、コードスニペットにMarkdownを含めない予定です。ただし、Markdownで文書化されたコードは、本書のGitHubレポで提供されます。さて、十分に文書化され、実行可能なCargoプロジェクトができたので、コンテキストに応じて異なる設定を実行できるように、パラメータを渡すことができるようにする必要があります。

### Cargoとの連携

プログラムを実行し、サードパーティモジュールを使用できるようになったので、コマンドライン入力を通じてRustプログラムと対話することができるようになりました。プログラムがコンテキストに応じた柔軟性を持つようにするには、プログラムにパラメータを渡すことができ、プログラムが実行されているパラメータを追跡できるようにする必要があります。これは、std（標準ライブラリ）識別子を使用して行うことができます。

```rust
use std::env;
fn main() {
    let args: Vec<String> = env::args().collect();
    println!("{:?}", args);
}
```

先のコードでは、プログラムに渡された引数をベクトルに集め、デバッグモードで引数を出力していることがわかります。次のコマンドを実行してみましょう。

```bash
cargo run one two three
```

先のコマンドを実行すると、次のようなプリントアウトが表示されます。

```
["target/debug/interacting_with_cargo", "one", "two", "three"]
```

ここで、argsベクトルには、私たちが渡した引数が含まれていることがわかります。他の多くの言語でも、コマンドラインからプログラムに渡される引数を受け入れるので、これは驚くべきことではありません。また、バイナリへのパスも含まれていることに注意しなければなりません。ここでは、interacting_with_cargoという別のプロジェクトを使っているため、target/debug/interacting_with_cargoというパスがあることも強調しておきます。また、コマンドライン引数から、デバッグモードで実行されていることがわかります。では、次のコマンドでプログラムのリリース版を実行してみましょう。

```bash
cargo run --release one two three
```

次のようなプリントを受け取ることになります。

```
["target/release/interacting_with_cargo", "one", "two", "three"]
```

先の出力から、--releaseは我々のベクトルにはないことがわかる。しかし、このことは、私たちが遊ぶための余分な機能を与えてくれる。例えば、コンパイルの種類によって異なるプロセスを実行したい場合がある。これは、次のようなコードで簡単に実現することができる。

```rust
let args: Vec<String> = env::args().collect();
let path: &str = &args[0];
if path.contains("/debug/") {
    println!("Debug is running");
}
else if path.contains("/release/") {
    println!("release is running");
}
else {
    panic!("The setting is neither debug or release");
}
```

しかし、先のシンプルな解決策は、パッチワークのようなものです。抽出するパスは、Cargoコマンドを実行している場合のみ一貫性があります。Cargoコマンドはビルド、コンパイル、ドキュメント作成には最適ですが、本番でそれらのファイルをすべて持ち運ぶのは意味がありません。実際、静的なバイナリを抽出して、それを完全に単体でDockerコンテナに包み、バイナリを直接実行することで、Dockerイメージのサイズを1.5GBから200MBに削減できるという利点があります。つまり、これは一見手っ取り早いように見えますが、私たちのアプリケーションをデプロイする際にコードを壊してしまうことにつながるのです。したがって、これが本番に到達してあなたが知らないということがないように、最後にパニックマクロを入れることが不可欠です。

これまでは、基本的なコマンドを渡していましたが、これでは役に立ちませんし、拡張性もありません。また、ユーザー向けのヘルプガイドを実装するために、多くの定型的なコードを書かなければならないでしょう。コマンドラインインターフェイスを拡張するために、プログラムに渡される引数を処理するためにclap crateに依存することができます。

```toml
[dependencies]
clap = "3.0.14"
```

コマンドラインインタフェースの理解を深めるために、いくつかのコマンドを取り込み、それをプリントアウトするだけのおもちゃのアプリケーションを開発することができます。そのためには、main.rsファイルの中でclap crateから必要なものを以下のコードでインポートする必要があります。

```rust
use clap::{Arg, App};
```

さて、次はアプリケーションの定義に移ります。

1. 私たちのアプリケーションは、次のコードでメイン関数にアプリケーションに関するメタデータを収容します。

```rust
fn main() {
    let app = App::new("booking")
        .version("1.0")
        .about("Books in a user")
        .author("Maxwell Flitton");
    ...
```

clapのドキュメントを見ると、App structに直接引数をバインドすることができます。しかし、これは醜く、きついバインドになる可能性があります。その代わりに、次のステップで別々に定義することにします。

2. 今回のおもちゃアプリでは、姓、名、年齢を取り込んでおり、以下のように定義することができる。

```rust
    let first_name = Arg::new("first name")
            .long("f")
            .takes_value(true)
            .help("first name of user")
            .required(true);
    let last_name = Arg::new("last name")
            .long("l")
            .takes_value(true)
            .help("first name of user")
            .required(true);
    let age = Arg::new("age")
            .long("a")
            .takes_value(true)
            .help("age of the user")
            .required(true);
```

引数を積み重ね続けることができることがわかります。今はまだ、何にも束縛されていません。さて、次のステップでは、アプリケーションにバインドして引数を渡せるようにします。

3. 入力のバインド、取得、解析は、以下のコードで実現できます。

```rust
    let app = app.arg(first_name).arg(last_name).arg(age);
    let matches = app.get_matches();
    let name = matches.value_of("first name")
            .expect("First name is required");
    let surname = matches.value_of("last name")
            .expect("Surname is required");
    let age: i8 = matches.value_of("age")
            .expect("Age is required").parse().unwrap();
    
    println!("{:?}", name);
    println!("{:?}", surname);
    println!("{:?}", age);
}
```

これで、コマンドライン引数の渡し方の実用例ができたので、次のコマンドを実行して、アプリケーションの表示方法を確認することができます。

```bash
cargo run -- --help  
```

真ん中の--before --helpは、Cargoに対して、--の後の引数をすべてclapに渡すように指示します。先のコマンドで、次のようなプリントアウトが得られます。

```
booking 1.0
Maxwell Flitton
Books in a user
USAGE:
    interacting_with_cargo --f <first name> --l <last name> 
                           --a <age>
OPTIONS:
        --a <age>           age of the user
        --f <first name>    first name of user
    -h, --help              Print help information
        --l <last name>     first name of user
    -V, --version           Print version information
```

先の出力では、コンパイルしたバイナリファイルを直接操作する方法を見ることができます。また、素敵なヘルプメニューも用意されています。Cargoと対話するためには、次のコマンドを実行する必要があります。

```bash
cargo run -- --f first --l second --a 32
```

先のコマンドで、次のようなプリントアウトがされます。

```
"first"
"second"
32
```

2つの文字列と1つの整数があるので、パージングがうまくいっていることがわかります。clapのようなクレートが有用な理由は、基本的に自己文書化されていることです。開発者はコードを見て、どのような引数が受け取られているのかを知り、その周りのメタデータを見ることができます。ユーザーは、helpパラメータを渡すだけで、入力に関するヘルプを得ることができます。このアプローチは、実行するコードに埋め込まれるため、ドキュメントが古くなるリスクを減らすことができます。コマンドライン引数を受け取る場合は、clapのようなクレートを使用することをお勧めします。さて、コマンドライン・インターフェースを構造化してスケーリングできるようにすることを検討しましたが、次のセクションでは、複数のファイルにわたるコードを構造化してスケーリングできるようにすることを検討します。
