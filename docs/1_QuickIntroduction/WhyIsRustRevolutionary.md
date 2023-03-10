## なぜRustは革命的なのか？

プログラミングでは、通常、スピードとリソース、開発スピードと安全性の間でトレードオフが発生します。C/C++などの低レベル言語は、高速なコード実行と最小限のリソース消費で、開発者がコンピュータをきめ細かく制御することができます。しかし、これは自由ではありません。手作業によるメモリ管理は、バグやセキュリティの脆弱性をもたらす可能性があります。その簡単な例が、バッファオーバーフロー攻撃です。これは、プログラマーが十分なメモリを割り当てなかった場合に発生します。例えば、バッファのサイズが15バイトしかなく、20バイトが送信された場合、余分な5バイトが境界を越えて書き込まれる可能性があります。攻撃者は、バッファが処理できる以上のバイトを渡すことで、これを悪用することができます。この場合、実行可能なコードが格納されている領域を独自のコードで上書きしてしまう可能性があります。メモリが正しく管理されていないプログラムを悪用する方法は他にもあります。脆弱性の増加に加えて、低レベル言語で問題を解決するためには、より多くのコードと時間が必要です。このため、C++ウェブフレームワークはウェブ開発において大きなシェアを占めることはありません。その代わりに、Python、Ruby、JavaScriptなどの高水準言語を使用するのが一般的です。このような言語を使うことで、開発者は安全かつ迅速に問題を解決することができます。

しかし、このメモリ安全性にはコストがかかることに注意しなければならない。これらの高級言語は一般に、定義されたすべての変数とそのメモリアドレスへの参照を記録しています。あるメモリ・アドレスを指す変数がなくなると、そのメモリ・アドレスのデータは削除される。このプロセスはガベージコレクションと呼ばれ、変数をクリーンアップするためにプログラムを停止しなければならないため、余分なリソースと時間を消費する。

Rustでは、コストのかかるガベージコレクション処理を行うことなく、メモリ安全性を確保することができます。Rustは、コンパイル時にボローチェッカーでチェックされる一連の所有権ルールによってメモリ安全性を確保します。これらのルールは、次のセクションで言及するクセのあるものです。このため、Rustは、真にパフォーマンスの高いコードで迅速かつ安全に問題を解決することができ、速度と安全性のトレードオフを解消することができます。

メモリの安全性

メモリ安全性とは、プログラムが常に有効なメモリを指すメモリポインタを持つという性質です。

より多くのデータ処理、トラフィック、複雑なタスクがウェブスタックに持ち込まれるようになり、ウェブフレームワークやライブラリの数が増えているRustは、今やウェブ開発の有力な選択肢となりました。このため、Rustのウェブ空間では、本当に驚くべき結果が得られています。2020年、Shimul Chowdhuryは、同じスペックで言語やフレームワークが異なるサーバに対して一連のテストを実施しました。その結果は、次の図に見ることができます。


図1.1 - Shimul Chowdhuryによる異なるフレームワークと言語の結果 (found at https://www.shimul.dev/posts/06-04-2020-benchmarking-flask-falcon-actix-web-rocket-nestjs/)

前図を見ると、言語やフレームワークのバリエーションがあることがわかります。しかし、RustフレームワークはActix WebとRocketで構成されていることに注目しなければなりません。これらのRustサーバーは、総リクエスト処理量やデータ転送量において、まったく別格の存在です。Golangなど他の言語も登場しましたが、Rustにはガベージコレクションがないため、Golangを凌駕することができました。これは、Jesse Howarth氏のブログ記事「Why Discord is switching from Go to Rust」で実証されており、以下のグラフが掲載されています。


図1.2 - Discordの所見⇒Golangはスパイキー、Rustはスムース（https://discord.com/blog/why-discord-is-switching-from-go-to-rust で発見。）

Golangがメモリの安全性を保つために実装していたガベージコレクションは、2分間のスパイクをもたらす結果となりました。これは、すべてにRustを使うべきだと言っているわけではありません。仕事に適した道具を使うのがベストプラクティスです。これらの言語には、それぞれ異なる利点があります。先ほどの図は、Rustの良さを表示したに過ぎません。

ガベージコレクションが必要ないのは、Rustがボローチェッカーを使った強制ルールでメモリの安全性を確保しているからです。さて、なぜRustでコーディングしたいのかを理解したところで、次節ではデータ型の見直しに移ります。
