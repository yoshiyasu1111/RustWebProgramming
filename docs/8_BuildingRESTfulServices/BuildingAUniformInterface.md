## 統一されたインターフェースの構築

統一されたインターフェースを持つことで、リソースをURLで一意に識別することができます。これにより、バックエンドエンドポイントとフロントエンドビューが切り離され、フロントエンドビューとバックエンドエンドポイントの間で衝突することなく、アプリを拡張することができます。バックエンドをフロントエンドから切り離すには、バージョンタグを使用します。URLエンドポイントにv1やv2などのバージョンタグが含まれている場合、その呼び出しがバックエンドのRustサーバにヒットしていることがわかります。Rustサーバを開発しているとき、APIコールの新しいバージョンに取り組みたいと思うことがあります。しかし、開発中のバージョンにユーザがアクセスできるようにはしたくありません。テストサーバに別のバージョンをデプロイしている間、ライブユーザがあるバージョンにアクセスできるようにするには、サーバの API バージョンを動的に定義する必要があります。本書でこれまでに得た知識があれば、config.ymlファイルにバージョン番号を定義し、それを読み込むだけでよいでしょう。しかし、リクエストのたびにconfig.ymlの設定ファイルを読み込まなければならなくなります。データベース接続プールをセットアップするときに、config.ymlファイルから接続文字列を一度読み込んだことを思い出してください。つまり、プログラムの全生涯にわたって存在することになります。私たちは、バージョンを一度定義し、その後、プログラムのライフサイクルのためにそれを参照したいと思います。直感的には、次の例のように、サーバーを定義する前にmain.rsファイルのmain関数でバージョンを定義し、wrap_fnの中でバージョンの定義にアクセスしたいかもしれません。

```rust
let outcome = "test".to_owned();
HttpServer::new(|| {
    . . .
    let app = App::new()
        .wrap_fn(|req, srv|{
            println!("{}", outcome);
            . . .
        }
    . . .
```

しかし、先のコードをコンパイルしようとすると、結果変数の寿命が十分でないため、失敗します。そこで、次のコードで結果変数を定数として変換することができます。

```rust
const OUTCOME: &str = "test";
HttpServer::new(|| {
    . . .
    let app = App::new()
        .wrap_fn(|req, srv|{
            println!("{}", outcome);
            . . .
        }
    . . .
```

先のコードは、寿命の問題なく実行されます。しかし、バージョンを読み込むとしたら、ファイルから読み込む必要があります。Rustでは、ファイルから読み込む場合、ファイルから読み込む変数の大きさがわかりません。したがって、ファイルから読み込む変数は文字列になります。ここで問題になるのは、文字列の確保はコンパイル時に計算できるものではないことです。そのため、main.rsファイルに直接バージョンを書き込むことになります。これは、ビルドファイルを使うことで実現できます。

備考

この問題ではビルドファイルを活用し、ビルドファイルの概念を教えることで、必要であればビルドファイルを使えるようにします。コードに定数をハードコーディングすることを止めるものは何もありません。

ここでは、Rustアプリケーションを実行する前に、1つのRustファイルが実行されます。このビルドファイルは、メインのRustアプリケーションをコンパイルしているときに自動的に実行されます。ビルドファイルの実行に必要な依存関係は、Cargo.tomlファイルのビルド依存関係セクションで、以下のコードで定義できます。

```toml
[package]
name = "web_app"
version = "0.1.0"
edition = "2021"
build = "build.rs"
[build-dependencies]
serde_yaml = "0.8.23"
serde = { version = "1.0.136", features = ["derive"] }
```

つまり、ビルドRustファイルは、アプリケーションのルートにあるbuild.rsファイルに定義されます。次に、[build-dependencies]セクションで、ビルドフェーズに必要な依存関係を定義します。これで依存関係が定義されたので、build.rsファイルは次のような形になります。

```rust
use std::fs::File;
use std::io::Write;
use std::collections::HashMap;
use serde_yaml;
fn main() {
  let file =
      std::fs::File::open("./build_config.yml").unwrap();
  let map: HashMap<String, serde_yaml::Value> =
      serde_yaml::from_reader(file).unwrap();
  let version =
      map.get("ALLOWED_VERSION").unwrap().as_str()
          .unwrap();
  let mut f =
      File::create("./src/output_data.txt").unwrap();
  write!(f, "{}", version).unwrap();
}
```

ここでは、YAMLファイルから読み取る必要があるものをインポートし、標準のテキストファイルに書き込む必要があることがわかります。このファイルは Web アプリケーションのルートにあり、config.yml ファイルと隣り合っています。そして、build_config.ymlファイルからALLOWED_VERSIONを抽出し、テキストファイルに書き込みます。これで、ビルドプロセスとbuild_config.ymlファイルから必要なものを定義したので、build_config.ymlファイルは次のような形にする必要があります。

```rust
ALLOWED_VERSION: v1
```

これで、ビルドの定義がすべて終わったので、build.rsファイルに記述したファイルを通して、バージョンのconstインスタンスを導入することができます。そのために、main.rsファイルを少し変更する必要があります。まず、次のコードでconstを定義します。

```rust
const ALLOWED_VERSION: &'static str = include_str!(
    "./output_data.txt");
HttpServer::new(|| {
. . .
```

そして、以下のコードでバージョンが許可されている場合にリクエストをパスしたとみなす。

```rust
HttpServer::new(|| {
. . .
let app = App::new()
.wrap_fn(|req, srv|{
    let passed: bool;
    if *&req.path().contains(&format!("/{}/",
                             ALLOWED_VERSION)) {
        passed = true;
    } else {
        passed = false;
    }
```

次に、以下のコードでエラーレスポンスとサービスコールを定義します。

```rust
. . .
let end_result = match passed {
    true => {
        Either::Left(srv.call(req))
    },
    false => {
        let resp = HttpResponse::NotImplemented()
            .body(format!("only {} API is supported",
                ALLOWED_VERSION));
        Either::Right(
            ok(req.into_response(resp).map_into_boxed_body())
        )
    }
};
. . .
```

これで、特定のバージョンをサポートしたアプリケーションをビルドして実行する準備ができました。アプリケーションを実行し、v2リクエストを行うと、次のようなレスポンスが返ってきます。


図8.4 - ブロックされたv2に対するPostmanの応答

バージョンガードが機能するようになったことが確認できます。これはまた、フロントエンドのビューにアクセスするためにReactアプリケーションを使用しなければならないことを意味し、またはフロントエンドAPIエンドポイントにv1を追加することができます。

さて、アプリを実行すると、フロントエンドが新しいエンドポイントで動作することが確認できます。これで、アプリの RESTful API 開発に一歩近づいたことになります。しかし、まだいくつかの目立った欠点があります。今のところ、別のユーザーを作成して、そのユーザーの下でログインすることができます。次のセクションでは、ステートレス方式でユーザーの状態を管理する方法を探ります。
