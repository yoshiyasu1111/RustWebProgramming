## クロージャを理解する

クロージャは基本的に関数ですが、名前を持たない匿名関数でもあります。つまり、クロージャを関数や構造体に渡すことができるのです。しかし、クロージャの受け渡しについて掘り下げる前に、空のRustプログラム（お好みでRustプレイグラウンドを使用できます）に次のコードで基本的なクロージャを定義して、クロージャについて調べてみましょう。

```rust
fn main() {
    let test_closure = |string_input| {
        println!("{}", string_input);
    };
    test_closure("test");
}
```

先のコードを実行すると、次のようなプリントアウトが得られます。

```
test
```

先の出力では、クロージャが関数のように動作していることがわかります。しかし、入力を定義するために中括弧を使う代わりに、パイプを使用しています。

先ほどのクロージャで、string_inputパラメータのデータ型を定義していないことにお気づきかもしれませんが、このコードはまだ実行されます。これは、パラメータのデータ型を定義する必要がある関数とは異なります。これは、関数がユーザーに公開される明示的なインターフェースの一部であるためです。コードが関数にアクセスできるのであれば、関数はコード内のどこからでも呼び出すことができます。一方、クロージャは寿命が短く、そのスコープにしか関係しない。このため、コンパイラはスコープ内でのクロージャの使用状況から、クロージャに渡される型を推測することができます。クロージャを呼び出すときに&strを渡しているので、コンパイラはstring_inputの型が&strであることを知っています。これは便利なことですが、クロージャは汎用的ではないことを知っておく必要があります。つまり、クロージャは具体的な型を持っているのです。例えば、クロージャを定義した後、次のようなコードを実行してみましょう。

```
    test_closure("test");
    test_closure(23);
```

以下のようなエラーが出ます。

```
7 |     test_closure(23);
  |                  ^^ expected `&str`, found integer
```

このエラーは、クロージャの最初の呼び出しによって、コンパイラに「&str」を期待していることが伝わり、2回目の呼び出しによってコンパイル処理が中断されるために発生します。

スコープが影響するのはクロージャだけではありません。クロージャは変数と同じスコープルールに従います。例えば、次のようなコードを実行してみようとしたとします。

```rust
fn main() {
    {
        let test_closure = |string_input| {
            println!("{}", string_input);
            };
    }
    test_closure("test");
}
```

クロージャを呼び出そうとすると、呼び出しのスコープに入らないので、コンパイルが拒否されるのです。これを考えると、クロージャには他のスコープルールが適用されると考えるのが正しいでしょう。例えば、次のようなコードを実行しようとしたら、どうなると思いますか？

```rust
fn main() {
    let another_str = "case";
    let test_closure = |string_input| {
        println!("{} {}", string_input, another_str);
    };
    test_closure("test");
}
```

次のような出力が得られると思ったら、その通りでしたね。

```
test case
```

関数とは異なり、クロージャは自分のスコープ内の変数にアクセスすることができます。そこで、クロージャを私たちが理解できるように単純化して説明すると、クロージャはスコープ内の動的変数のようなもので、計算を実行するために呼び出すものです。

次のコードのように、moveを利用することで、クロージャで使用する外部変数を所有することができます。

```rust
let test_closure = move |string_input| {
    println!("{} {}", string_input, another_str);
};
```

ここで定義したクロージャではmoveを利用しているため、test_closureがanother_strの所有権を取得し、test_closure宣言後にanother_str変数を利用することはできない。

クロージャを関数に渡すこともできますが、関数を他の関数に渡すこともできることに注意する必要があります。次のようなコードで、関数を他の関数に渡すことを実現できます。

```rust
fn add_doubles(closure: fn(i32) -> i32, one: i32, two: i32) -> i32 {
    return closure(one) + closure(two)
}

fn main() {
    let closure = |int_input| {
        return int_input * 2
    };
    let outcome = add_doubles(closure, 2, 3);
    println!("{}", outcome);
}
```

先のコードでは、渡された整数を2倍にして返すクロージャを定義していることがわかります。そして、これをadd_doubles関数にfn(i32)->i32という表記で渡しています（関数ポインタとして知られています）。クロージャに関しては、次のような特徴を持つものを実装することができます。

- Fn:不変的に変数を借用する
- FnMut。変数を相互に借用する
- FnOnce: 変数の所有権を持ち、一度だけ呼び出すことができる。

add_doubles関数に、先の特徴のいずれかを実装したクロージャを渡すには、次のようなコードを記述します。

```rust
fn add_doubles(closure: Box<dyn Fn(i32) -> i32>, one: i32, two: i32) -> i32 {
    return closure(one) + closure(two)
}

fn main() {
    let one = 2;
    let closure = move |int_input| {
        return int_input * one
    };
    let outcome = add_doubles(Box::new(closure), 2, 3);
    println!("{}", outcome);
}
```

ここで、クロージャ関数のパラメータがBox<dyn Fn(i32) -> i32>シグネチャを持つことがわかります。これは、add_doubles関数が、i32を受け取り、i32を返すFn特性を実装したクロージャを受け入れていることを意味します。Box構造体はスマートポインターで、クロージャのサイズがコンパイル時にわからないため、ヒープにクロージャを配置しています。また、クロージャを定義する際にmoveを使用していることもわかります。これは、クロージャの外側にある1つの変数を使うためです。クロージャの定義にmoveを使用したため、1つの変数は十分な寿命を持たない可能性があるため、クロージャがその所有権を取得します。

クロージャについて学んだことを念頭に置いて、次のコードでサーバーアプリケーションのメイン関数をもう一度見てみましょう。

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
            .route("/say/hello", web::get().to(|| async { "Hello Again!" }))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

先のコードでは、HttpServer::new関数を使用してHttpServerを構築した後、実行していることがわかります。現在わかっていることは、App構造体を返すクロージャを渡していることです。クロージャについて知っていることを踏まえれば、このコードで行うことにもっと自信を持つことができます。クロージャがApp構造体を返すのであれば、基本的にクロージャの中で好きなことをすることができます。このことを念頭に置いて、次のコードでプロセスに関するより多くの情報を得ることができます。

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        println!("http server factory is firing");
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
            .route("/say/hello", web::get().to(|| async { "Hello Again!" }))
    })
    .bind("127.0.0.1:8080")?
    .workers(3)
    .run()
    .await
}
```

先のコードでは、クロージャが発火していることを伝えるためにprint文が追加されていることがわかります。また、workersという別の関数も追加しています。これは、サーバーの作成に使用されるワーカーの数を定義できることを意味します。また、クロージャの中でサーバーファクトリが起動していることをプリントアウトしています。先のコードを実行すると、次のようなプリントアウトが得られます。

```bash
    Finished dev [unoptimized + debuginfo] target(s) in 
    2.45s
     Running `target/debug/web_app`
http server factory is firing
http server factory is firing
http server factory is firing
```

先の結果は、クロージャが3回発射されたことを物語っています。workersの数を変更すると、クロージャの起動回数と直接的な関係があることがわかります。もしworkers関数を省略した場合は、システムのコア数に応じてクロージャが起動されることになります。次節では、このワーカーがサーバープロセスにどのように適合しているかを探ります。

App structの構築に関するニュアンスを理解したところで、いよいよプログラムの構造の主な変更点である非同期プログラミングについて見ていきましょう。
