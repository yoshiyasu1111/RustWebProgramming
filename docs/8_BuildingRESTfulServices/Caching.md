## キャッシング

キャッシングとは、フロントエンドにデータを保存して再利用することです。これにより、バックエンドへのAPI呼び出し回数を減らし、待ち時間を短縮することができます。利点がはっきりしているため、すべてをキャッシュしたくなることもあるでしょう。しかし、いくつか考慮すべき点があります。

同時並行性は明らかな問題です。データが古くなり、バックエンドに間違った情報を送ることで混乱やデータの破損につながる可能性があります。また、セキュリティ上の懸念もあります。あるユーザーがログアウトし、別のユーザーが同じコンピューターでログインした場合、2番目のユーザーが最初のユーザーのアイテムにアクセスできてしまう危険性があります。このため、いくつかのチェックが必要です。正しいユーザーがログインしている必要があり、また、キャッシュされたデータが一定期間を過ぎてアクセスされた場合、データを更新するためにGETリクエストが行われるように、データにタイムスタンプを付ける必要があります。

私たちのアプリケーションはかなりロックダウンされています。ログインしていない限り、何もアクセスできません。私たちのアプリケーションでキャッシュできる主な処理は、GET items呼び出しです。バックエンドでアイテムリストの状態を編集する他のすべての呼び出しは、更新されたアイテムを返します。これを考慮すると、私たちのキャッシュ機構は次のようになります。


図8.7 - キャッシングの考え方

図のループは、ページを更新する際に何度でも実行することができます。しかし、これはあまり良いアイデアではないかもしれません。ユーザーがキッチンにいるときにスマホでアプリケーションにログインしてリストを更新したとすると、ユーザーがパソコンに戻って作業をするときに、パソコンでページを更新してリストを更新してしまうという問題が発生します。このキャッシュシステムは、バックエンドに送信される古いデータにユーザーをさらすことになります。タイムスタンプを参照することで、このような事態が起こるリスクを減らすことができます。タイムスタンプがカットオフより古い場合、ユーザーがページを更新したときにデータを更新するために、別のAPIコールを行うことになります。

キャッシュ・ロジックに関しては、すべてfront_end/src/App.jsファイルのgetItems関数で実装されます。getItems関数は、次のようなレイアウトになります。

```js
getItems() {
  . . .
  if (difference <= 120) {
      . . .
  }
  else {
      axios.get("http://127.0.0.1:8000/v1/item/get",
          {headers: {"token": localStorage
                      .getItem("token")}}).then(response => {
              . . .
              })
          }).catch(error => {
          if (error.response.status === 401) {
              this.logout();
          }
      });
  }
}
```

ここでは、最後にキャッシュされた項目と現在時刻との時間差が120秒（2分）以下でなければならないとしています。時間差が2分以下であれば、キャッシュからデータを取得することになります。しかし、時間差が2分以上であれば、APIバックエンドにリクエストを行います。もし、不正なレスポンスが返ってきた場合は、ログアウトすることになります。まず、このgetItems関数で、アイテムがキャッシュされた日付を取得し、以下のコードで当時と現在の差を計算します。

```js
let cachedData = Date.parse(localStorage
                            .getItem("item-cache-date"));
let now = new Date();
let difference = Math.round((now - cachedData) / (1000));
```

時差が2分の場合、次のコードでローカルストレージからデータを取得し、そのデータで状態を更新します。

```js
let pendingItems =
    JSON.parse(localStorage.getItem("item-cache-data-pending"));
let doneItems =
    JSON.parse(localStorage.getItem("item-cache-data-
                                     done"));
let pendingItemsCount = pendingItems.length;
let doneItemsCount = doneItems.length;
this.setState({
  "pending_items": this.processItemValues(pendingItems),
  "done_items": this.processItemValues(doneItems),
  "pending_items_count": pendingItemsCount,
  "done_items_count": doneItemsCount
})
```

ここでは、ローカルストレージが文字列データしか扱えないため、ローカルストレージのデータをパースする必要があります。ローカルストレージは文字列しか扱えないため、APIリクエストを行う際にローカルストレージに挿入するデータを以下のコードで文字列化する必要があります。

```js
let pending_items = response.data["pending_items"]
let done_items = response.data["done_items"]
localStorage.setItem("item-cache-date", new Date());
localStorage.setItem("item-cache-data-pending",
                      JSON.stringify(pending_items));
localStorage.setItem("item-cache-data-done",
                      JSON.stringify(done_items));
this.setState({
  "pending_items": this.processItemValues(pending_items),
  "done_items": this.processItemValues(done_items),
  "pending_items_count":
      response.data["pending_item_count"],
  "done_items_count": response.data["done_item_count"]
})
```

アプリケーションを実行すると、APIコールは1回しか行われません。2分経過する前にアプリケーションを更新すると、フロントエンドがキャッシュからすべてのアイテムをレンダリングしているにもかかわらず、新しいAPI呼び出しがないことがわかります。しかし、アイテムを作成、編集、または削除した後、2分が経過する前にページを更新すると、ビューが以前の古い状態に戻ってしまうことがわかります。これは、作成、編集、削除されたアイテムも以前の状態に戻るが、ローカルストレージに保存されていないためである。これは、handleReturnedState関数を次のコードで更新することで対処できます。

```js
handleReturnedState = (response) => {
  let pending_items = response.data["pending_items"]
  let done_items = response.data["done_items"]
  localStorage.setItem("item-cache-date", new Date());
  localStorage.setItem("item-cache-data-pending",
                        JSON.stringify(pending_items));
  localStorage.setItem("item-cache-data-done",
                        JSON.stringify(done_items));
  this.setState({
     "pending_items":this.processItemValues(pending_items),
     "done_items": this.processItemValues(done_items),
     "pending_items_count":response
         .data["pending_item_count"],
      "done_items_count": response.data["done_item_count"]
  })
}
```

ここで、私たちはそれを手に入れました。データをキャッシュして再利用することで、バックエンドAPIが過剰にヒットするのを防いでいるのです。これは他のフロントエンドの処理にも適用できます。例えば、顧客バスケットをキャッシュし、ユーザーがチェックアウトするときに使用することができます。

これで、私たちのシンプルなウェブサイトはウェブアプリに一歩近づいたことになります。しかし、キャッシングをもっと使うようになると、フロントエンドの複雑さが増すことを認識しなければなりません。私たちのアプリケーションでは、ここでキャッシングを停止します。今、私たちのアプリケーションでは、残りの1時間、これ以上手を加える必要はありません。しかし、もう1つ簡単に説明しなければならない概念があります。
