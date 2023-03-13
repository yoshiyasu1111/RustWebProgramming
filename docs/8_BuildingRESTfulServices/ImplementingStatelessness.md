## ステートレスを実現する

ステートレスとは、サーバーがクライアントセッションに関する情報を一切保存しないことです。この利点は単純明快です。セッション情報をクライアント側に保存することで、サーバー側のリソースを解放するため、アプリケーションをより簡単に拡張できるようになります。

また、コンピューティングアプローチをより柔軟に行うことができるようになります。例えば、アプリケーションの人気が爆発的に高まったとします。その結果、2つのコンピューティング・インスタンスまたはサーバーでアプリケーションを起動し、ロードバランサーが両方のインスタンスにバランスよくトラフィックを誘導するようにしたいと思うかもしれません。情報がサーバーに保存されていると、ユーザーは一貫性のない体験をすることになります。

あるコンピューティングインスタンスでセッションの状態を更新しても、別のリクエストをしたときに、古いデータを持つ別のコンピューティングインスタンスにぶつかるかもしれない。このことを考えると、ステートレス化は、クライアントにすべてを保存することだけで実現できるわけではありません。データベースがアプリのコンピューティングインスタンスに依存していない場合、以下の図のように、このデータベースにデータを保存することも可能です。


図8.5 - ステートレスアプローチ

ご覧のように、私たちのアプリはすでにステートレスです。フロントエンドではユーザーIDをJWTで保存し、PostgreSQLデータベースにはユーザーデータモデルとToDoアイテムを保存しています。しかし、アプリケーションにRust構造体を格納したいと思うかもしれません。たとえば、サーバーを叩くリクエストの数をカウントする構造体を構築することができます。図8.5を参照すると、構造体をサーバにローカルに保存することはできません。そこで、Redisに構造体を保存し、下図のような処理を行います。


図8.6 「Redisに構造体を保存するための手順

PostgreSQLとRedisの違いについて

Redisはデータベースですが、PostgreSQLのデータベースとは異なります。Redisはキーバリューストアに近いです。また、Redisはデータがメモリ上にあるため、高速です。RedisはPostgreSQLのようにテーブルを管理し、それらが互いにどのように関連しているかを完全に把握しているわけではありませんが、Redisには利点があります。Redisは、リスト、セット、ハッシュ、キュー、チャネルといった便利なデータ構造をサポートしています。また、Redisに挿入するデータには有効期限を設定することができます。また、Redisではデータの移行を処理する必要がありません。このため、Redisは、素早くアクセスする必要があるが、永続性にはあまりこだわらないデータをキャッシュするための理想的なデータベースと言えます。チャネルとキューについては、Redisはサブスクライバーとパブリッシャー間の通信を促進するのにも理想的です。

以下の手順を踏むことで、図8.6の処理を実現することができる。

1. Docker用のRedisサービスを定義しました。
2. Rustの依存関係を更新します。
3. Redis接続のための設定ファイルを更新する。
4. Redisデータベースで保存・読み込み可能なカウンタ構造体を構築する。
5. 各リクエストにカウンターを実装する。

それでは、各ステップを詳しく見ていきましょう。

1. RedisのDockerサービスを立ち上げる際には、標準のRedisコンテナを標準のポートで使用する必要があります。Redisサービスを実装したら、docker-compose.ymlファイルは現在の状態になっているはずです。

```yaml
version: "3.7"
services:
  postgres:
    container_name: 'to-do-postgres'
    image: 'postgres:11.2'
    restart: always
    ports:
      - '5433:5432'
    environment:
      - 'POSTGRES_USER=username'
      - 'POSTGRES_DB=to_do'
      - 'POSTGRES_PASSWORD=password'
  redis:
      container_name: 'to-do-redis'
      image: 'redis:5.0.5'
      ports:
        - '6379:6379'
```

これで、ローカルマシン上でRedisサービスとデータベースサービスが実行されていることが確認できます。Redisを実行できるようになったので、次のステップで依存関係を更新する必要があります。

図8.6を参照すると、Rust構造体をRedisに挿入する前に、バイトにシリアライズする必要があります。これらの手順を念頭に置いて、Cargo.tomlファイルには以下の依存関係が必要です。

```toml
[dependencies]
. . .
redis = "0.21.5"
```

Redisデータベースへの接続には、redis crateを使用しています。これで依存関係が定義されたので、次のステップで設定ファイルの定義を開始できます。

3. config.ymlファイルには、Redisデータベース接続用のURLを追加する必要があります。本書のこの時点では、config.ymlファイルは次のような形になっているはずです。

```yaml
DB_URL: postgres://username:password@localhost:5433/to_do
SECRET_KEY: secret
EXPIRE_MINUTES: 120
REDIS_URL: redis://127.0.0.1/
```

REDIS_URLパラメータには、ポート番号を追加していません。これは、Redisサービスの標準ポートである6379を使用するためで、ポートを定義する必要はありません。これで、Redisに接続するための構造体を定義するためのデータがすべて揃いましたので、次のステップでこれを実行します。

4. src/counter.rsファイルにCounter構造体を定義します。まず、以下のものをインポートする必要があります。

```rust
use serde::{Deserialize, Serialize};
use crate::config::Config;
```

5. RedisのURLを取得するためにConfigインスタンスを使用し、バイトへの変換を可能にするためにDeserializeとSerializeトレイトを使用します。Counter構造体は次のような形をしています。

```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct Counter {
    pub count: i32
}
```

さて、Counter構造体にすべての特徴を定義したので、次のコードで操作に必要な関数を定義する必要があります。

```rust
impl Counter {
    fn get_redis_url() -> String {
        . . .
    }
    pub fn save(self) {
        . . .
    }
    pub fn load() -> Counter {
        . . .
    }
}
```

7. 前述の関数を定義することで、RedisデータベースへのCounter構造体のロードと保存が可能になります。get_redis_url関数を作成する場合、次のような形になることは驚くことではありません。

```rust
fn get_redis_url() -> String {
    let config = Config::new();
    config.map.get("REDIS_URL")
              .unwrap().as_str()
              .unwrap().to_owned()
}
```

8. RedisのURLがわかったので、次のコードでCounter構造体を保存します。

```rust
pub fn save(self) -> Result<(), redis::RedisError> {
    let serialized = serde_yaml::to_vec(&self).unwrap();
    let client = match redis::Client::open(
                     Counter::get_redis_url()) {
        Ok(client) => client,
        Err(error) => return Err(error)
    };
    let mut con = match client.get_connection() {
        Ok(con) => con,
        Err(error) => return Err(error)
    };
    match redis::cmd("SET").arg("COUNTER")
                           .arg(serialized)
                           .query::<Vec<u8>>(&mut con) {
        Ok(_) => Ok(()),
        Err(error) => Err(error)
    }
}
```

9. ここでは、Counter構造体をVec<u8>にシリアライズできることを確認しています。次に、Redisのクライアントを定義し、"COUNTER "というキーの下に、シリアライズされたCounter構造体を挿入します。Redisにはもっと多くの機能がありますが、本章ではRedisをスケーラブルなインメモリ・ハッシュマップとして考えることで、Redisを活用できます。ハッシュマップの概念を念頭に置いて、RedisデータベースからCounter構造体を取得するにはどうしたらいいと思いますか？COUNTER」をキーにGETコマンドを使い、次のようなコードでデシリアライズするのです。

```rust
pub fn load() -> Result<Counter, redis::RedisError> {
    let client = match redis::Client::open(
                     Counter::get_redis_url()){
        Ok(client) => client,
        Err(error) => return Err(error)
    };
    let mut con = match client.get_connection() {
        Ok(con) => con,
        Err(error) => return Err(error)
    };
    let byte_data: Vec<u8> = match redis::cmd("GET")
                                 .arg("COUNTER")
                                 .query(&mut con) {
        Ok(data) => data,
        Err(error) => return Err(error)
    };
    Ok(serde_yaml::from_slice(&byte_data).unwrap())
}
```

これで、Counter構造体を定義しました。これで、次のステップでmain.rsファイルに実装するものがすべて揃いました。

10. リクエストが来るたびにカウントを1つ増やすということでは、main.rsファイル内で以下のコードを実行する必要があります。

```rust
. . .
mod counter;
. . .
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    . . .
    let site_counter = counter::Counter{count: 0};
    site_counter.save();
    HttpServer::new(|| {
        . . .
        let app = App::new()
            .wrap_fn(|req, srv|{
                let passed: bool;
                let mut site_counter = counter::
                                       Counter::load()
                                       .unwrap();
                site_counter.count += 1;
                println!("{:?}", &site_counter);
                site_counter.save();
                . . .
```

11. ここでは、Counterモジュールを定義していることがわかります。サーバーをスピンアップする前に、新しいCounter構造体を作成し、Redisに挿入する必要があります。次に、Redisからカウンターを取得し、カウントを増加させ、リクエストごとにそれを保存します。
これで、サーバーを起動すると、サーバーにリクエストが来るたびにカウンターが増加していることが確認できます。プリントアウトは以下のような感じになるはずです。

```rust
Counter { count: 1 }
Counter { count: 2 }
Counter { count: 3 }
Counter { count: 4 }
```

もう1つのストレージオプションを統合したことで、このアプリケーションは基本的に私たちが望むように機能するようになりました。もし今すぐアプリケーションを出荷したいのであれば、Dockerでビルドを構成し、データベースとNGINXを備えたサーバーにデプロイすることを止めるものは何もありません。

並行処理に関する問題

2つのサーバーが同時にカウンターをリクエストすると、リクエストの取りこぼしが発生する危険性があります。カウンターの例は、Redisにシリアル化された構造体を格納する方法を示すために検討されました。Redisデータベースで単純なカウンターを実装する必要があり、同時実行が懸念される場合は、INCRコマンドを使用することが推奨されます。INCRコマンドは、Redisデータベースで選択したキーの下の数値を1つ増やし、その結果として新しく増えた数値を返します。Redisデータベースでカウンタが増加するため、同時並行性の問題のリスクを軽減することができます。

しかし、常に追加できるものがあります。次のセクションでは、ロギング要求について調査します。
