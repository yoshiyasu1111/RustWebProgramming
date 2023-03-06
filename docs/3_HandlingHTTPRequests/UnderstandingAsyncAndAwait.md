## asyncとawaitを理解する

asyncとawaitの構文は、前のセクションで説明したのと同じ概念を管理するものですが、いくつかのニュアンスがあります。単にスレッドを生成するのではなく、フューチャーを作成し、必要な時に必要な分だけ操作します。

コンピュータサイエンスにおいて、未来とは、未処理の計算のことです。これは、まだ結果が得られていない状態ですが、呼び出したり待ったりすると、未来に計算の結果が入力されます。別の表現では、未来はまだ準備ができていない値を表現する方法であるとも言えます。その結果、フューチャーは正確にはスレッドではありません。実際、スレッドはその可能性を最大限に引き出すために、フューチャーを使うことができます。例えば、いくつかのネットワーク接続があるとします。それぞれのネットワーク接続に対して、個別のスレッドを用意することができます。これは、すべての接続を順次処理するよりも優れています。遅いネットワーク接続は、それ自身が処理されるまで、他の速い接続の処理を妨げることになり、結果として全体として処理時間が遅くなります。しかし、すべてのネットワーク接続に対してスレッドをスピンアップすることは、無料ではありません。その代わり、ネットワーク接続ごとに未来を用意することができます。このネットワーク接続は、フューチャーの準備ができたときに、スレッドプールからスレッドで処理することができます。したがって、同時接続が多いWebプログラミングにおいて、futureが使われる理由がわかります。

フューチャーは、約束、遅延、延期とも呼ばれることがあります。フューチャーを探求するために、新しいCargoプロジェクトを作成し、Cargo.tomlファイルに作成されたフューチャーを利用することにします。

```toml
[dependencies]
futures = "0.3.21"
```

先行するcrateがインストールされているので、以下のコードを使ってmain.rsで必要なものをインポートすることができます。

```rust
use futures::executor::block_on;
use std::{thread, time};
```

async構文を使うだけで、futureを定義することができます。block_on関数は、定義したfutureが実行されるまで、プログラムをブロックします。これで、次のコードでdo_something関数を定義することができます。

```rust
async fn do_something(number: i8) -> i8 {
    println!("number {} is running", number);
    let two_seconds = time::Duration::new(2, 0);
    thread::sleep(two_seconds);
    return 2
}
```

do_something関数は、基本的にコードに書いてある通り、何番かを出力し、2秒間スリープして、整数を返します。しかし、これを直接呼び出すと、i8は得られない。代わりに、do_something関数を直接呼び出すと、Future<Output = i8>が得られます。次のコードで、main関数内でfutureを実行し、時間を計ることができます。

```rust
fn main() {
    let now = time::Instant::now();
    let future_one = do_something(1);
    let outcome = block_on(future_one);
    println!("time elapsed {:?}", now.elapsed());
    println!("Here is the outcome: {}", outcome);
}
```

先のコードを実行すると、次のようなプリントアウトが得られます。

```
number 1 is running
time elapsed 2.00018789s
Here is the outcome: 2
```

これは期待されるものです。しかし、次のコードでblock_on関数を呼び出す前に余分なsleep関数を入力するとどうなるかを見てみましょう。

```rust
fn main() {
    let now = time::Instant::now();
    let future_one = do_something(1);
    let two_seconds = time::Duration::new(2, 0);
    thread::sleep(two_seconds);
    let outcome = block_on(future_one);
    println!("time elapsed {:?}", now.elapsed());
    println!("Here is the outcome: {}", outcome);
}
```

以下のようなプリントを得ることができます。

```
number 1 is running
time elapsed 4.000269667s
Here is the outcome: 2
```

このように、block_on関数を使ってエクゼキュータを適用するまでは、未来は実行されないことがわかります。

同じ関数の中で後で実行できる未来が欲しいだけかもしれないので、これは少し手間がかかるかもしれません。これを実現するには、async/await構文が必要です。例えば、以下のようなコードで、do_something関数を呼び出し、main関数内でawait構文を使って終わるまでコードをブロックすることができます。

```rust
let future_two = async {
    return do_something(2).await
};
let future_two = block_on(future_two);
println!("Here is the outcome: {:?}", future_two);
```

asyncブロックが行うことは、futureを返すことです。このブロックの中で、do_something関数をawait式を使って、do_something関数が解決されるまでasyncブロックをブロックして呼び出すのです。そして、future_two futureにblock_on関数を適用します。

先ほどのコードブロックを見ると、do_something関数を呼び出してblock_on関数に渡すというたった2行のコードでできてしまうので、少し過剰に感じるかもしれません。この場合、過剰ではありますが、フューチャーの呼び出し方についてより柔軟性を持たせることができます。例えば、次のようなコードでdo_something関数を2回呼び出し、それをreturnとして足し合わせることができます。

```rust
let future_three = async {
    let outcome_one = do_something(2).await;
    let outcome_two = do_something(3).await;
    return outcome_one + outcome_two
};
let future_outcome = block_on(future_three);
println!("Here is the outcome: {:?}", future_outcome);
```

先のコードをmain関数に追加すると、次のようなプリントアウトが得られます。

```
number 2 is running
number 3 is running
Here is the outcome: 4
```

先の出力は期待通りの結果ですが、これらのフューチャーは順次実行されるため、このコードのブロックの合計時間は4秒強になることが分かっています。joinを使えば、これを高速化できるかもしれません。joinは同時に実行することでスレッドをスピードアップさせることができることを見てきました。Joinが先物のスピードアップに役立つことは理にかなっています。まず、次のコードでjoinマクロをインポートする必要があります。

```rust
use futures::join
```

これで先物にjoinを利用することができ、以下のコードで実装のタイミングを計ることができます。

```rust
let future_four = async {
    let outcome_one = do_something(2);
    let outcome_two = do_something(3);
    let results = join!(outcome_one, outcome_two);
    return results.0 + results.1
};
let now = time::Instant::now();
let result = block_on(future_four);
println!("time elapsed {:?}", now.elapsed());
println!("here is the result: {:?}", result);
```

先のコードでは、joinマクロが結果のタプルを返し、そのタプルをアンパックして同じ結果を得ていることがわかります。しかし、このコードを実行すると、望む結果が得られるにもかかわらず、未来の実行速度が上がらず、4秒強で止まっていることがわかります。これは、asyncタスクを使ってfutureが実行されていないためです。フューチャーの実行を高速化するためには、非同期タスクを使用する必要があります。そのためには、次のようなステップを踏んでください。

1. 必要な先物を作る。
2. それをベクターに入れる。
3. ベクトルをループし、ベクトル内の各未来に対してタスクをスピンオフさせる。
4. 非同期タスクを結合し、ベクトルを合計する。

これを視覚的にマッピングすると、次の図のようになります。

図3.2-複数のフューチャーを同時に実行するための手順

すべてのフューチャーを同時に結合するには、別のcrateを使用して、async_std crateを使用して独自の非同期結合関数を作成する必要があります。このクレートをCargo.tomlファイルに以下のコードで定義します。

```
async-std = "1.11.0"
```

async_stdクレートが手に入ったので、図3.2のアプローチを実行するために必要なものを、main.rsファイルの先頭で次のコードでインポートすることができます。

```rust
use std::vec::Vec;
use async_std;
use futures::future::join_all;
```

main関数では、次のようなコードで未来を定義することができるようになりました。

```rust
let async_outcome = async {
    // 1.
    let mut futures_vec = Vec::new();
    let future_four = do_something(4);
    let future_five = do_something(5);
    // 2.
    futures_vec.push(future_four);
    futures_vec.push(future_five);
    // 3. 
    let handles = futures_vec.into_iter().map(
    async_std::task::spawn).collect::<Vec<_>>();
    // 4.
    let results = join_all(handles).await;
    return results.into_iter().sum::<i8>();
};
```

ここでは、先物を定義し（1）、それをベクトルに追加しています（2）。次に、into_iter関数を使ってvector内のfutureをループします。そして、async_std::task::spawnを使って、各未来に対してスレッドを起動します。これはstd::task::spawnと似ています。では、なぜこのような余計な手間をかける必要があるのでしょうか？ベクトルをループして、各タスクに対してスレッドを生成すればいいのです。ここでの違いは、async_std::task::spawn関数が、同じスレッドで非同期タスクをスピンオフしていることです。したがって、同じスレッドで両方のフューチャーを同時に実行することになります。次に、すべてのハンドルを結合して、これらのタスクの終了を待ち、すべてのスレッドの合計を返します。これでasync_outcome未来が定義できたので、次のコードで実行し、時間を計ることができます。]]

```rust
let now = time::Instant::now();
let result = block_on(async_outcome);
println!("time elapsed for join vec {:?}", now.elapsed());
println!("Here is the result: {:?}", result);
```

追加したコードを実行すると、次のような追加プリントが表示されます。

```
number 4 is running
number 5 is running
time elapsed for join vec 2.007713458s
Here is the result: 4
```

うまくいってますね。同じスレッドで2つの非同期タスクを同時に実行させることに成功しました。その結果、両方のフューチャーが2秒強で実行されました

このように、Rustではスレッドや非同期タスクの生成は簡単です。しかし、スレッドや非同期タスクに変数を渡すことはできないので、注意が必要です。Rustの借用機構はメモリ安全性を確保します。スレッドにデータを渡す際には、余分な手順を踏まなければなりません。スレッド間のデータ共有の一般的な概念についてさらに議論することは、今回のWebプロジェクトには適していません。しかし、どのような型がデータを共有できるかを簡単に説明することができます。

- std::sync::Arc: スレッドが外部データを参照できるようにするタイプです。

```rust
use std::sync::Arc;
use std::thread;
let names = Arc::new(vec!["dave", "chloe", "simon"]);
let reference_data = Arc::clone(&names);
    
let new_thread = thread::spawn(move || {
    println!("{}", reference_data[1]);
});
```

- std::sync::Mutexです。この型は、スレッドが外部データを変異させることを可能にします。

```rust
use std::sync::Mutex;
use std::thread;
let count = Mutex::new(0);
    
let new_thread = thread::spawn(move || {
    count.lock().unwrap() += 1;
});
```

ここのスレッド内部では、ロックの結果をデリファレンスし、それをアンラップし、変異させる。注意しなければならないのは、共有状態はロックが保持されているときにしかアクセスできないことです。

これで非同期プログラミングは十分理解できたので、Webプログラミングに戻りましょう。並行処理については、1冊の本でカバーできる内容です。「さらに読む」セクションで、そのうちの1冊を参照しました。Rustの非同期プログラミングの知識が、Actix Webサーバーの理解にどのように影響するのかを確認するために、今はWeb開発におけるRustの探求に戻らなければなりません。
