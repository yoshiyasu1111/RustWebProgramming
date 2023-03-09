## HTMLにJavaScriptを注入する

このセクションを終えると、JavaScriptを使ってRustサーバを呼び出し、ToDoアイテムの追加、編集、削除を行うことができる、それほどきれいではありませんが、十分に機能するメインビューが完成します。ただし、削除APIエンドポイントを追加していないことを思い出してください。HTMLにJavaScriptを注入するには、次の手順を実行する必要があります。

1. delete item APIエンドポイントを作成します。
2. JavaScriptの読み込み機能を追加し、HTMLデータ内のJavaScriptタグを、メインアイテムのRustビューで読み込んだJavaScriptデータと置き換える。
3. HTMLファイルにJavaScriptタグを追加し、JavaScriptでコンポーネントを参照できるようにHTMLコンポーネントにIDを追加する。
4. JavaScriptでToDoアイテムのレンダリング関数を作成し、IDを使用してHTMLにバインドします。
5. JavaScriptでAPIコール関数を作成し、バックエンドとやり取りする。
6. JavaScriptで、取得、削除、編集、作成の各機能を構築し、ボタンに使用させる。
では、詳しく見ていきましょう。

詳しく見ていきましょう。

### 削除エンドポイントの追加

削除APIエンドポイントを追加するのは、もう簡単なはずです。もしそうしたいのであれば、このビューを自分で実装してみることをお勧めします。このプロセスにはもう慣れているはずです。

1. もし悩んでいるのであれば、以下のサードパーティーの依存関係をviews/to_do/delete.rsファイルにインポートすることで実現できます。

```rust
use actix_web::{web, HttpResponse};
use serde_json::value::Value;
use serde_json::Map;
```

これらは新しいものではないので、使い慣れているはずですし、どこを活用すればいいのかもわかっているはずです。

2. 次に、次のコードで構造体と関数をインポートする必要があります。

```rust
use crate::to_do::{to_do_factory, enums::TaskStatus};
use crate::json_serialization::{to_do_item::ToDoItem, to_do_items::ToDoItems};
use crate::processes::process_input;
use crate::jwt::JwToken;
use crate::state::read_file;
```

ここでは、to_doモジュールを使って、ToDo項目を作成していることがわかります。json_serializationモジュールで、ToDoItemを受け取り、ToDoItemsを返していることがわかります。そして、process_input関数で項目の削除を実行しています。また、このページを訪れた人が、アイテムを削除することは避けたい。そのため、JwToken構造体が必要です。最後に、read_file関数でアイテムの状態を読み取ります。

3. これで必要なものが揃ったので、次のコードで削除ビューを定義します。

```rust
pub async fn delete(to_do_item: web::Json<ToDoItem>, token: JwToken) -> HttpResponse {
        . . .
}

ここでは、ToDoItemをJSONとして受け入れ、ビューにJwTokenを添付して、ユーザーがアクセスするための認証が必要であることを確認できます。この時点では、JwTokenがメッセージを添付しているだけです。JwTokenの認証ロジックは、第7章「ユーザーセッションの管理」で管理することになります。

4. deleteビューの内部では、以下のコードでJSONファイルを読み込むことで、ToDoアイテムの状態を取得することができます。

```rust
let state: Map<String, Value> = read_file("./state.json");
```

5. そして、このタイトルのアイテムがステートにあるかどうかをチェックします。ない場合は、not found HTTP responseを返します。もしあれば、アイテムを構築するためにタイトルとステータスが必要なので、ステータスを渡します。このチェックとステータスの抽出は、次のコードで実現できます。

```rust
let status: TaskStatus;
match &state.get(&to_do_item.title) {
    Some(result) => {
        status = TaskStatus::from_string(result.as_str().unwrap().to_string());
    }
    None => {
        return HttpResponse::NotFound().json(format!("{} not in state", &to_do_item.title))
    }
}
```

6. これで、ToDoアイテムのステータスとタイトルがわかったので、アイテムを作成し、deleteコマンドを付けてprocess_input関数に渡すことができます。これで、JSONファイルからアイテムが削除されます。

```rust
let existing_item = to_do_factory(to_do_item.title.as_str(), status.clone());
process_input(existing_item, "delete".to_owned(), &state);
```

7. ToDoItems構造体にResponder traitを実装し、ToDoItems::get_state()関数はJSONファイルからアイテムを入力したToDoItems構造体を返すことを思い出してください。したがって、deleteビューから次のようなreturn文が返されます。

```rust
return HttpResponse::Ok().json(ToDoItems::get_state())
```

8. deleteビューが定義されたので、それをsrc/views/to_do/mod.rsファイルに追加すると、次のようなビューファクトリーになります。

```rust
mod create;
mod get;
mod edit;
mod delete;
use actix_web::web::{ServiceConfig, post, get, scope};
pub fn to_do_views_factory(app: &mut ServiceConfig) {
    app.service(
        scope("v1/item")
            .route("create/{title}", post().to(create::create))
            .route("get", get().to(get::get))
            .route("edit", post().to(edit::edit))
            .route("delete", post().to(delete::delete))
    );
}
```

9. to_do_views_factoryを素早く検査することで、ToDoアイテムの管理に必要なすべてのビューがあることがわかります。このモジュールをアプリケーションから取り出して、別のアプリケーションに挿入するとしたら、何を削除し、何を追加したかを即座に確認することができます。

削除ビューがアプリケーションに完全に統合されたので、2番目のステップであるJavaScriptのロード機能を構築することに進みます。

### JavaScriptのローディング機能を追加する

さて、すべてのエンドポイントの準備ができたので、メインアプリのビューを再確認してみましょう。前のセクションでは、`<script>`セクションのJavaScriptが、1つの大きな文字列の一部であるにもかかわらず、動作することを確認しました。そこで、このビューでは、HTMLファイルの`<script>`セクションに{{JAVASCRIPT}}タグを記述した文字列として、HTMLファイルを読み込みます。次に、JavaScriptファイルを文字列として読み込み、{{JAVASCRIPT}}タグをJavaScriptファイルからの文字列に置き換えます。最後に、views/app/items.rsファイル内のbodyで完全な文字列を返します。

```rust
pub async fn items() -> HttpResponse {
    let mut html_data = read_file("./templates/main.html");
    let javascript_data = read_file("./javascript/main.js");
    html_data = html_data.replace("{{JAVASCRIPT}}", &javascript_data);

    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(html_data)
}
```

### HTML内にJavaScriptタグを追加する

前のステップのアイテム関数から、ルートにJavaScriptという新しいディレクトリを構築する必要があることがわかります。また、その中にmain.jsという名前のファイルを作成する必要があります。このアプリビューの変更に伴い、templates/main.htmlファイルにも次のコードを追加して変更する必要があります。

```html
<body>
    <h1>Done Items</h1>
    <div id="doneItems"></div>
    <h1>To Do Items</h1>
    <div id="pendingItems"></div>
    <input type="text" id="name" placeholder="create to do
     item">
    <button id="create-button" value="Send">Create</button>
</body>
<script>
    {{JAVASCRIPT}}
</script>
```

エンドポイントは、保留中の項目と完了した項目を返すことを思い出してください。IDが「doneItems」のdivには、APIコールで完了したToDo項目が挿入されます。

次に、APIコールで取得した保留アイテムを、IDが「pendingItems」のdivに挿入します。その後、テキストとボタンを含む入力を定義する必要があります。これは、ユーザーが新しいアイテムを作成するためのものです。

### レンダリングJavaScript関数の構築

HTMLが定義できたので、次はjavascript/main.jsファイルにロジックを定義します。

1. 最初に作る関数は、メインページのすべてのToDo項目をレンダリングするものです。これは、javascript/main.jsファイルのコードの中で最も複雑な部分であることに注意する必要があります。私たちは基本的に、HTMLコードを書くJavaScriptコードを書いているのです。後ほど、Reactアプリを作成するセクションで、Reactフレームワークを使用してこれを行う必要性を置き換える予定です。今のところ、アイテムのリストを作成するレンダー関数を構築することにします。各アイテムは、HTMLで次のような形式をとります。

```html
<div>
    <div>
        <p>learn to code rust</p>
        <button id="edit-learn-to-code-rust">
            edit
        </button>
    </div>
</div>
```

ToDo項目のタイトルが段落のHTMLタグの中に入れ子になっていることがわかります。そして、ボタンがあります。HTMLタグのidプロパティは一意でなければならないことを思い出してください。そこで、このIDを、ボタンで何をするかということと、ToDo項目のタイトルをもとに構成します。このidプロパティに、イベントリスナーを使ってAPIコールを行う関数をバインドすることができるようになります。

2. レンダー関数を作成するには、レンダリングする項目、実行する処理の種類（編集または削除）、これらの項目をレンダリングするHTMLのセクションの要素ID、および各ToDo項目ボタンに結合する関数を渡す必要があります。この関数の概要は、次のコードで定義されています。

```rust
function renderItems(items, processType, elementId, processFunction) {
    . . .
}
```

3. renderItems関数の内部では、まずHTMLを構築し、以下のコードでToDo項目をループさせることができます。

```rust
let itemsMeta = [];
let placeholder = "<div>"
for (let i = 0; i < items.length; i++) {
    . . .
}
    placeholder += "</div>"
    document.getElementById(elementId).innerHTML = placeholder;
```

ここでは、各ToDo項目に対して生成するHTMLのメタデータを収集するための配列を定義しています。これはitemsMetaという変数に格納され、後ほどrenderItems関数でイベントリスナーを使ってprocessFunctionを各ToDoアイテムのボタンにバインドするために使用されます。次に、1つのプロセスに対してすべてのToDo項目を収容するHTMLをplaceholder変数で定義します。ここでは、divタグから始めます。そして、項目ごとにループしてデータをHTMLに変換し、最後にdivタグを閉じてHTMLを完成させます。その後、placeholderと呼ばれる構築されたHTML文字列をinnerHTMLに挿入しています。innerHTMLの位置は、構築したToDo項目を表示するページです。

4. ループの中では、次のコードで個々のToDo項目のHTMLを構築する必要があります。

```rust
let title = items[i]["title"];
let placeholderId = processType + "-" + title.replaceAll(" ", "-");
placeholder += "<div>" + title + "<button " + 'id="' + placeholderId + '">' + processType +'</button>' + "</div>";
itemsMeta.push({"id": placeholderId, "title": title});
```

ここでは、ループしている項目から項目のタイトルを抽出しています。次に、イベント・リスナーにバインドするために使用するアイテムのIDを定義します。タイトルとIDを定義したので、プレースホルダのHTML文字列にタイトルを持つdivを追加します。さらに、placeholderIdを持つボタンを追加し、最後にdivを追加して終了します。HTML文字列への追加が、;で終了していることがわかります。そして、placeholderIdとtitleをitemsMeta配列に追加し、後で使用するようにしています。

5. 次に、itemsMetaをループさせ、以下のコードでイベントリスナーを作成します。

```rust
   . . .
   placeholder += "</div>"
   document.getElementById(elementId).innerHTML = placeholder;
   for (let i = 0; i < itemsMeta.length; i++) {
        document
            .getElementById(itemsMeta[i]["id"])
            .addEventListener("click", processFunction);
    }
}
```

さて、processFunctionは、ToDo項目の隣に作成したボタンがクリックされた場合に起動します。この関数でアイテムがレンダリングされましたが、APIコール関数でバックエンドからアイテムを取得する必要があります。これについては、これから見ていきます。

### APIコールJavaScript関数の構築

レンダー関数ができたので、APIコール関数を見ましょう。

1. まず、javascript/main.jsファイルにAPIコール関数を定義する必要があります。この関数は、APIコールのエンドポイントであるURLを受け取ります。また、POST、GET、PUTのいずれかの文字列であるメソッドを受け取ります。次に、リクエスト・オブジェクトを定義する必要があります。

```rust
function apiCall(url, method) {
    let xhr = new XMLHttpRequest();
    xhr.withCredentials = true;
```

2. そして、apiCall関数内にイベントリスナーを定義し、呼び出しが終了したら返されたJSONでToDo項目をレンダリングする必要があります。

```rust
xhr.addEventListener('readystatechange', function() {
    if (this.readyState === this.DONE) {
        renderItems(
            JSON.parse(this.responseText)["pending_items"], 
            "edit",
            "pendingItems",
            editItem);
        renderItems(
            JSON.parse(this.responseText)["done_items"],
            "delete",
            "doneItems",
            deleteItem);
    }
});
```

ここでは、templates/main.html ファイルで定義した ID を渡していることがわかります。また、APIコールからのレスポンスも渡しています。また、editem関数を渡していますが、これは、保留中のアイテムに付随するボタンがクリックされ、そのアイテムが完了アイテムになったときに、edit関数を実行することを意味しています。これを踏まえて、完了したアイテムのボタンがクリックされると、deleteItem関数が実行されます。とりあえず、apiCall関数を作り続けます。

3. この後、editItem関数とdeleteItem関数を構築する必要があります。また、apiCall関数が呼ばれるたびに、アイテムがレンダリングされることも分かっています。

イベントリスナーを定義したので、APIコールオブジェクトにメソッドとURLを準備し、ヘッダーを定義し、必要なときにいつでも送信できるようにリクエストオブジェクトを返さなければなりません。

```rust
    xhr.open(method, url);
    xhr.setRequestHeader('content-type', 
        'application/json');
    xhr.setRequestHeader('user-token', 'token');
    return xhr
}
```

さて、apiCall関数を使ってアプリケーションのバックエンドへの呼び出しを行い、API呼び出し後のアイテムの新しい状態でフロントエンドを再レンダリングすることができます。これで、最後のステップに進むことができます。ここでは、ToDoアイテムの作成、取得、削除、編集機能を実行する関数を定義します。

### ボタン用のJavaScript関数を作る

このヘッダーは、バックエンドでハードコーディングされているアクセプタントークンをハードコーディングしているだけであることに注意してください。第7章ユーザーセッションの管理で、authヘッダーを適切に定義する方法を説明します。APIコール関数が定義されたので、次はeditem関数に移ります。

```rust
function editItem() {
    let title = this.id.replaceAll("-", " ").replace("edit ", "");
    let call = apiCall("/v1/item/edit", "POST");
    let json = {
        "title": title,
        "status": "DONE"
    };
    call.send(JSON.stringify(json));
}
```

ここでは、イベントリスナーが属するHTMLセクションに、これを経由してアクセスできることがわかります。editを削除し、-をスペースに切り替えると、ToDoアイテムのIDがToDoアイテムのタイトルに変換されることがわかる。次に、apiCall関数を使用して、エンドポイントとメソッドを定義します。replace関数の "edit "の文字列にはスペースがあることに注意してください。このスペースがあるのは、edit文字列の後のスペースも削除する必要があるからです。このスペースを削除しないと、バックエンドに送信され、アプリケーションのバックエンドがJSONファイルのアイテムのタイトルの隣にスペースを持たないため、エラーになります。エンドポイントとAPI呼び出しメソッドが定義されたら、タイトルを辞書に渡し、ステータスをdoneとして渡します。これは、保留中のアイテムをdoneに切り替えることが分かっているからです。これが完了したら、JSONボディを含むAPIコールを送信します。

さて、同じ手法でdeleteItem関数も使えるようにします。

```rust
function deleteItem() {
    let title = this.id.replaceAll("-", " ").replace("delete ", "");
    let call = apiCall("/v1/item/delete", "POST");
    let json = {
        "title": title,
        "status": "DONE"
    };
    call.send(JSON.stringify(json));
}
```

ここでも、replace関数の "delete "文字列の中にスペースがあります。これで、レンダリング処理は完全に処理されたことになります。編集と削除の関数と、レンダリング関数を定義しました。次に、ページが最初に読み込まれたときに、ボタンをクリックすることなく、アイテムを読み込ませる必要があります。これは、簡単なAPIコールで実現できます。

```rust
function getItems() {
    let call = apiCall("/v1/item/get", 'GET');
    call.send()
}
getItems();
```

ここでは、GETメソッドでAPIコールを行い、それを送信しているだけであることがわかります。また、getItems関数が関数の外で呼び出されていることに注意してください。これは、ビューがロードされたときに一度だけ実行されます。

長い間コーディングをしてきましたが、あと少しで完成です。あとは、テキスト入力とボタンの作成機能を定義するだけです。これは、シンプルなイベント・リスナーとcreateエンドポイントのAPIコールで管理することができます。

```rust
document.getElementById("create-button")
        .addEventListener("click", createItem);
function createItem() {
    let title = document.getElementById("name");
    let call = apiCall("/v1/item/create/" + title.value, "POST");
    call.send();
    document.getElementById("name").value = null;
}
```

また、テキスト入力の値をnullに設定する詳細も追加しました。inputをnullに設定したのは、ユーザーが、作成されたばかりの古いアイテムのタイトルを削除することなく、別のアイテムを入力できるようにするためです。このアプリのメインビューを叩くと、次のような出力が得られます。



図5.5 レンダリングされたToDo項目が表示されたメインページ

さて、フロントエンドが思い通りに動くかどうかを確認するために、次のステップを実行します。

1. 洗濯完了の項目の横にある削除ボタンを押す。
2. 朝食にシリアルを食べる」と入力し、「作成」をクリックする。
3. 朝食にラーメンを食べる」と入力し、［作成］をクリックします。
4. 朝食にラーメンを食べる」項目の編集をクリックします。

以上の手順で、以下のような結果が得られるはずです。

図5.6-前述した手順が完了した後のメインページ

これで、完全に機能するウェブアプリが完成しました。すべてのボタンが機能し、リストが即座に更新されます。しかし、見た目はあまりきれいではありません。間隔がなく、すべてが白黒で表示されています。これを改善するには、HTMLファイルにCSSを組み込む必要がありますが、これは次のセクションで説明します。
