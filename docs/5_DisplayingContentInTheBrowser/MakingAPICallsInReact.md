## ReactでAPIを呼び出す

これで基本的なアプリケーションが動作するようになったので、バックエンドへのAPIコールを実行し始めることができます。このため、主にfront_end/src/App.jsファイルに焦点を当てます。Rustアプリケーションのアイテムをフロントエンドに入力できるように、アプリケーションを構築していきます。まず、package.jsonファイルのdependenciesに以下を追加する必要があります。

```json
"axios": "^0.26.1"
```

そして、以下のコマンドを実行します。

```bash
$ npm install
```

これにより、追加の依存関係がインストールされます。次に、front_end/src/App.js ファイルを開き、次のコードで必要なものをインポートします。

```js
import React, { Component } from 'react';
import axios from 'axios';
```

アプリクラスの継承にはComponentを、バックエンドへのAPI呼び出しにはaxiosを使用する予定です。さて、以下のコードでAppクラスを定義し、状態を更新することができます。

```js
class App extends Component {
  state = {
      "pending_items": [],
      "done_items": [],
      "pending_items_count": 0,
      "done_items_count": 0
  }
}
export default App;
```

ここでは、自作のフロントエンドと同じ構造になっています。これは、Rustサーバーのget itemsビューから返されるデータでもあります。さて、どのようなデータを扱うのかがわかったので、次のステップを実行することができます。

1. Appクラス内に、Rustサーバーから関数を取得する関数を作成します。
2. この関数がAppクラスがマウントされたときに実行されるようにします。
3. Appクラス内に、Rustサーバーから返されたアイテムをHTMLに処理する関数を作成します。
4. Appクラス内に、前述のコンポーネントをすべてレンダリングしてフロントエンドに表示する関数を作成し、終了します。
5. Rustサーバーが他のソースからの呼び出しを受信できるようにします。

これらのステップに入る前に、Appクラスのアウトラインが次のような形になっていることを確認しておきます。

```js
class App extends Component {
 
  state = {
      . . .
  }
  // makes the API call
  getItems() {
      . . .
  }
  // ensures the API call is updated when mounted
  componentDidMount() {
      . . .
  }
  // convert items from API to HTML 
  processItemValues(items) {
      . . .
  }
  // returns the HTML to be rendered
  render() {
    return (
        . . .
    )
  }
}
```

これで、APIコールを行う関数に着手することができます。

1. Appクラスの内部では、getItems関数が次のようなレイアウトをとります。

```js
axios.get("http://127.0.0.1:8000/v1/item/get", {headers: {"token": "some_token"}})
    .then(response => {
        let pending_items = response.data["pending_items"]
        let done_items = response.data["done_items"]
        this.setState({
              . . .
        })
    });
```

ここでは、URLを定義します。そして、ヘッダにトークンを追加します。今はまだRustサーバでユーザセッションを設定していないため、単純な文字列をハードコードします。そして、これを閉じます。axios.getはプロミスなので、.thenを使用する必要があります。.then括弧の中のコードは、データが返されたときに実行されます。この括弧の中で、必要なデータを抽出し、this.setState関数を実行します。this.setState関数は、Appクラスの状態を更新する関数です。ただし、this.setStateを実行すると、Appクラスのrender関数も実行され、ブラウザが更新されます。このthis.setState関数の内部では、以下のコードを渡しています。

```json
"pending_items": this.processItemValues(pending_items),
"done_items": this.processItemValues(done_items),
"pending_items_count": response.data["pending_item_count"],
"done_items_count": response.data["done_item_count"]
```

これでgetItemsが完成し、バックエンドからアイテムを取得することができるようになりました。さて、定義ができたので、次はこれを確実に実行する必要があります。

2. AppクラスがロードされたときにgetItems関数が発生し、状態が更新されるようにするためには、以下のコードを使用します。

```js
componentDidMount() {
  this.getItems();
}
```

getItemsは、Appコンポーネントがマウントされた直後に実行されるため、これは簡単です。つまり、componentDidMount関数でthis.setStateを呼び出しているのです。これにより、ブラウザが画面を更新する前に余分なレンダリングがトリガーされます。renderが2回呼び出されたとしても、ユーザーには中間状態は見えません。これは、React Componentクラスから継承した多くの関数の1つです。ページが読み込まれると同時にデータを読み込むことができたので、次のステップに進むことができます：読み込んだデータを処理します。

3. Appクラス内のprocessItemValues関数では、アイテムを表すJSONオブジェクトの配列を取り込み、HTMLに変換する必要がありますが、これは次のコードで実現できます。

```js
processItemValues(items) {
  let itemList = [];
  items.forEach((item, index)=>{
      itemList.push(
          <li key={index}>{item.title} {item.status}</li>
      )
  })
  return itemList
}
```

ここでは、項目をループしてli HTML要素に変換し、空の配列に追加して、埋まったら返すだけです。getItems関数でstateに入れる前に、processItemValue関数でデータを処理することを忘れないでください。これで、ステートにすべてのHTMLコンポーネントが揃ったので、render関数でページ上に配置する必要があります。

4. Appクラスでは、render関数はHTMLコンポーネントを返すだけです。これには余分なロジックは使っていません。以下のように返すことができます。

```html
<div className="App">
<h1>Done Items</h1>
<p>done item count: {this.state.done_items_count}</p>
{this.state.done_items}
<h1>Pending Items</h1>
<p>pending item count: {this.state.pending_items_count}</p>
{this.state.pending_items}
</div>
```

ここで、私たちの状態が直接参照されていることがわかります。これは、本章の前半で採用した手動での文字列操作から大きく変わった点です。Reactを使用することで、よりクリーンで、エラーのリスクを減らすことができます。フロントエンドでは、バックエンドへの呼び出しを伴うレンダリング処理が動作するはずです。しかし、Rustサーバーは、Reactアプリケーションからのリクエストを別のアプリケーションであることを理由にブロックします。これを修正するために、次のステップに進む必要があります。  

5. 今現在、私たちのRustサーバーは、サーバーへのリクエストをブロックします。これはCORS（Cross-Origin Resource Sharing：クロスオリジンリソース共有）に起因しています。CORSはデフォルトで同じオリジンからのリクエストを許可しているため、以前はこれといった問題はありませんでした。生のHTMLを書いてRustサーバーから配信していたときは、同じオリジンからリクエストが来ていたのです。しかし、Reactアプリケーションでは、リクエストは異なるオリジンからやってきます。これを修正するには、以下のコードでCargo.tomlファイルにCORSを依存関係としてインストールする必要があります。

```toml
actix-cors = "0.6.1"
```

src/main.rsファイルでは、次のコードでCORSをインポートする必要があります。

```rust
use actix_cors::Cors;
```

ここで、サーバーの定義の前にCORSポリシーを定義し、ビューの設定の直後に以下のコードでCORSポリシーをラップする必要があります。

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let cors = Cors::default().allow_any_origin()
                                  .allow_any_method()
                                  .allow_any_header();
        let app = App::new()
            .wrap_fn(|req, srv| {
                println!("{}-{}", req.method(), req.uri());
                let future = srv.call(req);
                async {
                    let result = future.await?;
                    Ok(result)
                }
            })
            .configure(views::views_factory)
            .wrap(cors);
        return app
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

これで、サーバーはReactアプリケーションからのリクエストを受け付ける準備が整いました。

備考

CORSポリシーを定義したとき、すべてのメソッド、ヘッダー、オリジンを許可するという表現になりました。しかし、次のようなCORSの定義では、より簡潔な表現が可能です。

let cors = Cors::permissive();

さて、アプリケーションをテストして、動作しているかどうかを確認しましょう。CargoでRustサーバを起動し、別の端末でReactアプリケーションを実行することで、これを行うことができます。これを実行すると、Reactアプリケーションがロードされたときに次のように表示されるはずです。


図5.11 - ReactアプリケーションがRustサーバーと最初に通信したときのビュー

これで、Rustアプリケーションへの呼び出しが期待通りに動作していることが確認できました。しかし、やっていることは、ToDoアイテムの名前とステータスをリストアップしているだけです。Reactが優れているのは、カスタムコンポーネントの構築です。つまり、ToDo項目ごとに独自の状態や関数を持つ個別のクラスを構築することができるのです。これについては、次のセクションで説明します。
