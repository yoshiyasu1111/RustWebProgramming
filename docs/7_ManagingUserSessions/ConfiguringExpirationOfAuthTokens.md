## 認証トークンの有効期限を設定する

ヘッダーのトークンでログインして取得した有効なトークンを使って、現在保護されているエンドポイントでAPIコールを実行しようとすると、不正なエラーが表示されます。print文を挿入すると、トークンのデコードに失敗して、次のようなエラーが発生します。

```
missing required claim: exp
```

これは、JwToken構造体の中にexpというフィールドが存在しないことを意味します。https://docs.rs/jsonwebtoken/latest/jsonwebtoken/fn.encode.html にある jsonwebtoken のドキュメントを参照すると、エンコード命令で exp について一切触れていないことがわかります。

```rust
use serde::{Deserialize, Serialize};
use jsonwebtoken::{encode, Algorithm, Header, EncodingKey};
#[derive(Debug, Serialize, Deserialize)]
struct Claims {
   sub: String,
   company: String
}
let my_claims = Claims {
    sub: "b@b.com".to_owned(),
    company: "ACME".to_owned()
};
// my_claims is a struct that implements Serialize
// This will create a JWT using HS256 as algorithm
let token = encode(&Header::default(), &my_claims,
&EncodingKey::from_secret("secret".as_ref())).unwrap();
```

ここでは、クレームについての言及がないことがわかります。しかし、トークンをデシリアライズしようとすると、jsonwebtokenクレートのデコード関数が自動的にexpフィールドを探し、トークンがいつ期限切れになるかを調べるということが起こっているのです。公式のドキュメントや少しわかりにくいエラーメッセージでは、何が起こっているのか理解するのに何時間も浪費してしまう可能性があるため、私たちはこのことを調査しています。このことを念頭に置いて、src/jwt.rsファイルに戻り、さらに書き直さなければなりませんが、これが最後であり、完全に書き直すわけではありません。まず、src/jwt.rsファイルにすでにあるものと一緒に、以下のものがインポートされていることを確認します。

```rust
. . .
use jsonwebtoken::{encode, decode, Header,
                   EncodingKey, DecodingKey,
                   Validation};
use chrono::Utc;
. . .
```

そして、次のコードでJwToken構造体にexpフィールドが書き込まれていることを確認できます。

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct JwToken {
    pub user_id: i32,
    pub exp: usize,
}
```

次に、JwToken構造体の新しいコンストラクタ・メソッドを書き直さなければなりません。新しい関数では、新しく鋳造されたJwToken構造体が何時期限切れになるかを定義する必要があります。開発者としては、タイムアウトにかかる時間を微調整したいかもしれません。Rustのコードを変更するたびに再コンパイルしなければならないので、タイムアウト時間を設定ファイルで定義することは理にかなっています。タイムアウトの分散を考慮すると、新しい関数は次のような形になります。

```rust
pub fn new(user_id: i32) -> Self {
    let config = Config::new();
    let minutes = config.map.get("EXPIRE_MINUTES")
                            .unwrap().as_i64().unwrap();
    let expiration = Utc::now()
    .checked_add_signed(chrono::Duration::minutes(minutes))
                            .expect("valid timestamp")
                            .timestamp();
    return JwToken { user_id, exp: expiration as usize };
}
```

分数を定義していることがわかります。次に、有効期限をusizeとして変換し、JwToken構造体を構築します。トークンのデコード時のエラーや、トークンの有効期限切れなど、返すエラーの種類を特定する必要があります。トークンのデコード時に発生するさまざまな種類のエラーは、次のコードで処理します。

```rust
pub fn from_token(token: String) -> Result<Self, String> {
    let key = DecodingKey::
              from_secret(JwToken::get_key().as_ref());
    let token_result = decode::<JwToken>(&token.as_str(),
                              &key,&Validation::default());
    match token_result {
        Ok(data) => {
            return Ok(data.claims)
        },
        Err(error) => {
            let message = format!("{}", error);
            return Err(message)
        }
    }
}
```

ここで、Optionを返していたのをResultに切り替えていることがわかります。Resultに切り替えたのは、FromRequest traitの実装にあるfrom_request関数で消化・処理できるメッセージを返すためです。from_request関数の残りのコードは同じです。変更点は、エラーが発生した場合にメッセージをチェックし、次のコードでフロントエンドに別のメッセージを返すことです。

```rust
fn from_request(req: &HttpRequest,
                _: &mut Payload) -> Self::Future {
    match req.headers().get("token") {
        Some(data) => {
            let raw_token = data.to_str()
                                .unwrap()
                                .to_string();
            let token_result = JwToken::
                        from_token(raw_token);
            match token_result {
                Ok(token) => {
                    return ok(token)
                },
                Err(message) => {
                    if message == "ExpiredSignature"
                                  .to_owned() {
                        return err(
                        ErrorUnauthorized("token expired"))
                    }
                    return err(
                    ErrorUnauthorized("token can't be decoded"))
                }
            }
        },
        None => {
            return err(
            ErrorUnauthorized(
            "token not in header under key 'token'"))
        }
    }
}
```

ニュアンスの異なるエラーメッセージを表示することで、フロントエンドのコードも対応し、適応できるようになります。フロントエンドでより具体的にエラーメッセージを表示することで、ユーザーがどこで間違ったのかを確認することができます。しかし、認証に関しては、あまり多くを語らないようにしましょう。これは、不正なアクセスを試みる悪意者を助けることにもなるからです。これで、ログインとログアウトのエンドポイントが動作するようになりました。また、必要なビューに対してトークンによる認証ができるようになりました。しかし、これは一般的なユーザがアプリケーションを操作する場合にはあまり役に立ちません。なぜなら、彼らはPostmanを使うことはまずないからです。したがって、次のセクションでは、フロントエンドにログイン/ログアウトエンドポイントを組み込む必要があります。
