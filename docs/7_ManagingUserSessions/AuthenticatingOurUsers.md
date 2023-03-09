## ユーザーを認証する

ユーザーの認証に関しては、HTTPリクエストのヘッダーからメッセージを抽出する構造体を構築しています。現在は、ヘッダーにユーザーに関するデータを格納することで、この抽出を実際に利用できる段階にあります。今現在、HTTPリクエストのヘッダーにユーザー名、ID、パスワードを保存して、それぞれのリクエストを認証できるようにすることを止めるものは何もありません。しかし、これはひどいやり方です。もし誰かがリクエストを傍受したり、これを容易にするためにブラウザに保存されたデータを手に入れたりしたら、アカウントは漏洩し、ハッカーはやりたい放題になってしまいます。その代わりに、次の図に示すように、データを難読化することにします。


図7.5「リクエストの認証のためのステップ

図7.5では、秘密鍵を使って、ユーザーに関する構造化データをバイト単位のトークンにシリアライズしていることがわかります。そして、そのトークンをユーザーに渡してブラウザに保存させる。ユーザが認可されたリクエストを行う場合、ユーザはリクエストのヘッダにトークンを送信する必要があります。次に私たちのサーバーは、秘密鍵を使用してトークンをデシリアライズし、ユーザーに関する構造化データに戻します。この処理を行うために使用されるアルゴリズムは、誰でも利用できる標準的なハッシュアルゴリズムです。したがって、トークンを安全に保管するために、私たちが定義する秘密鍵があります。このアプリケーションで図7.5のような処理を行うには、JwToken構造体を含むsrc/jwt.rsファイルの大半を書き換える必要があります。その前に、Cargo.tomlの依存関係を次のコードで更新しておく必要があります。

```rust
[dependencies]
. . .
chrono = {version = "0.4.19", features = ["serde"]}
. . .
jsonwebtoken = "8.1.0"
```

chrono crateにserdeの機能を追加し、jsonwebtoken crateを追加していることがわかります。JwToken構造体を再構築するために、src/jwt.rsファイルに以下をインポートする必要があります。

```rust
use actix_web::dev::Payload;
use actix_web::{Error, FromRequest, HttpRequest};
use actix_web::error::ErrorUnauthorized;
use futures::future::{Ready, ok, err};
use serde::{Deserialize, Serialize};
use jsonwebtoken::{encode, decode, Algorithm, Header,
                   EncodingKey, DecodingKey, Validation};
use chrono::{DateTime, Utc};
use chrono::serde::ts_seconds;
use crate::config::Config;
```

リクエストとレスポンスの処理を可能にするために、actix_web の traits と structs をインポートしていることがわかります。次に、futures をインポートして、HTTP リクエストがビューに到達する前にその傍受を処理できるようにします。次に、serdeとjsonwebtokenをインポートして、トークンとの間でデータのシリアライズとデシリアライズを可能にします。次に、トークンが鋳造されたときにログを記録するために、chrono crateをインポートします。また、シリアライズのためのキーが必要ですが、これは設定ファイルから取得するため、Config構造体をインポートします。これで、必要なtraitと構造体をすべてインポートできたので、次のコードでトークン構造体を書くことができます。

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct JwToken {
    pub user_id: i32,
    #[serde(with = "ts_seconds")]
    pub minted: DateTime<Utc>
}
```

ここでは、ユーザーの ID と、トークンが作成された日時を取得していることが確認できます。また、datetimeフィールドをどのようにシリアライズするかを示すために、serdeマクロでmintedフィールドを装飾しています。これでトークンに必要なデータが揃ったので、次のコードでシリアライズ関数を定義することができます。

```rust
impl JwToken {
    pub fn get_key() -> String {
        . . .
    }
    pub fn encode(self) -> String {
        . . .
    }
    pub fn new(user_id: i32) -> Self {
        . . .
    }
    pub fn from_token(token: String) -> Option<Self> {
        . . .
    }
}
```

先行する各機能が何をするのか、以下の箇条書きで説明することができます。

- get_keyを取得します。config.ymlファイルからシリアライズ、デシリアライズ用の秘密鍵を取得する。
- encodeする。JwToken structのデータをトークンとしてエンコードする。
- new: 新しい JwToken 構造体を作成する。
- from_token:トークンからJwToken構造体を作成します。デシリアライズに失敗した場合は、Noneを返します（デシリアライズに失敗することがあるため）。

上記の関数を構築すれば、JwToken構造体は適切なタイミングでトークンを処理できるようになります。次のコードでget_key関数を具体化します。

```rust
pub fn get_key() -> String {
    let config = Config::new();
    let key_str = config
        .map
        .get("SECRET_KEY")
        .unwrap()
        .as_str()
        .unwrap();

    return key_str.to_owned()
}
```

ここでは、キーをconfigファイルから読み込んでいることがわかります。したがって、config.ymlファイルにキーを追加する必要があり、その結果、ファイルは次のようになります。

```
DB_URL: postgres://username:password@localhost:5433/to_do
SECRET_KEY: secret
```

もし私たちのサーバーが本番環境であれば、もっと良い秘密鍵があるはずです。しかし、ローカルで開発する場合は、これで問題ないでしょう。さて、設定ファイルからキーを取り出したので、次のコードでエンコード関数を定義することができます。

```rust
pub fn encode(self) -> String {
    let key = EncodingKey::from_secret(JwToken::get_key().as_ref());
    let token = encode(&Header::default(), &self,&key).unwrap();

    return token
}
```

ここでは、設定ファイルのシークレットキーを使ってエンコード・キーを定義していることがわかります。このキーを使用して、JwToken構造体のデータをトークンにエンコードし、それを返します。これでJwToken構造体をエンコードできるようになったので、必要なときに新しいJwToken構造体を作成する必要がありますが、これは次の新しい関数で実現できます。

```rust
pub fn new(user_id: i32) -> Self {
    let timestamp = Utc::now();

    return JwToken { user_id, minted: timestamp };
}
```

コンストラクタで、JwTokenがいつ造幣されたかを知ることができます。これは、ユーザー・セッションを管理するのに役立ちます。例えば、トークンの年齢が適切と思われる閾値を超えた場合、再ログインを強制することができます。

あとはfrom_token関数で、以下のコードを使ってトークンからデータを抽出するだけです。

```rust
pub fn from_token(token: String) -> Option<Self> {
    let key = DecodingKey::from_secret(JwToken::get_key().as_ref());
    let token_result = decode::<JwToken>(
            &token,
            &key,
            &Validation::new(Algorithm::HS256)
    );

    match token_result {
        Ok(data) => {
            Some(data.claims)
        },
        Err(_) => {
            return None
        }
    }
}
```

ここでは、デコードキーを定義し、それを使ってトークンをデコードします。そして、data.claimsを使用してJwTokenを返します。これで、JwToken構造体の作成、トークンへのエンコード、トークンからの抽出ができるようになりました。あとは、ビューがロードされる前のHTTPリクエストのヘッダーから、次のようなアウトラインで抽出するだけです。

```rust
impl FromRequest for JwToken {
    type Error = Error;
    type Future = Ready<Result<JwToken, Error>>;
    fn from_request(req: &HttpRequest,
                    _: &mut Payload) -> Self::Future {
        . . .
    }
}
```

データベース接続のためのFromRequest traitと、JwToken構造体のための前回の実装を、これまで複数回繰り返してきました。from_request関数の内部では、以下のコードでヘッダーからトークンを抽出しています。

```rust
match req.headers().get("token") {
    Some(data) => {
        . . .
    },
    None => {
        let error = ErrorUnauthorized(
                    "token not in header under key 'token'"
                    );
        return err(error)
    }
}
```

トークンがヘッダーにない場合は、ErrorUnauthorizedを直接返し、ビューの呼び出しを完全に回避します。ヘッダーからトークンを取り出すことができれば、次のコードで処理することができます。

```rust
Some(data) => {
    let raw_token = data.to_str().unwrap().to_string();
    let token_result = JwToken::from_token(raw_token);
    match token_result {
        Some(token) => {
            return ok(token)
        },
        None => {
            let error = ErrorUnauthorized("token can't be decoded");
            return err(error)
        }
    }
},
```

ここでは、ヘッダーから抽出した生のトークンを文字列に変換しています。次に、トークンをデシリアライズして、JwToken構造体にロードします。ただし、偽のトークンが提供されたためにこれが失敗した場合は、ErrorUnauthorizedエラーを返します。これで認証は完全に機能するようになりました。しかし、次の図に示すように、有効なトークンを持っていないため、何もすることができなくなります。


図7.6 「認証ブロックのリクエスト

次のセクションでは、保護されたエンドポイントとの対話を可能にするために、ログインAPIエンドポイントを構築します。
