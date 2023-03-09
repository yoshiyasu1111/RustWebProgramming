## Rustを使ったHTML、CSS、JavaScriptのサービス提供

前章では、すべてのデータをJSONの形で返しました。このセクションでは、ユーザーが見ることのできるHTMLデータを返すことにします。このHTMLデータには、ユーザーが前章で定義したAPIエンドポイントと対話し、ToDo項目を作成、編集、削除できるようにするためのボタンとフォームを用意します。そのためには、次のような構造をとる独自のアプリビューモジュールを構成する必要があります。

```
views
├── app
│   ├── items.rs
│   └── mod.rs
```

### 基本的なHTMLを提供する

items.rsファイルでは、ToDo項目を表示するメインビューを定義する予定です。しかし、その前に、items.rsファイルでHTMLを返すことができる最も簡単な方法を探っておく必要があります。

```rust
use actix_web::HttpResponse;

pub async fn items() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body("<h1>Items</h1>")
}
```

ここでは、HTMLコンテンツタイプと<h1>Items</h1>のボディを持つHttpResponse構造体を返すだけにしています。アプリにHttpResponseを渡すには、app/views/mod.rsファイルに以下のようにファクトリーを定義する必要があります。

```rust
use actix_web::web;

mod items;

pub fn app_views_factory(app: &mut web::ServiceConfig) {
    app.route("/", web::get().to(items::items));
}
```

ここでは、サービスを構築する代わりに、単にアプリケーションのルートを定義していることがわかります。これは、ここがランディングページであるためです。もし、ルートではなくサービスを定義しようとすると、プレフィックスがないとサービスのビューを定義することができません。

app_views_factory を定義したら、views/mod.rs ファイルでこれを呼び出すことができます。ただし、その前に、views/mod.rs ファイルの先頭で app モジュールを定義する必要があります。

```rust
mod app;
```

アプリモジュールを定義したら、同じファイル内のviews_factory関数でアプリファクトリーを呼び出すことができます。

```rust
app::app_views_factory(app);
```

HTMLサービングビューがアプリの一部となったので、これを実行してブラウザでホームURLを呼び出すと、次のような出力が得られます。


図5.1 - 最初にレンダリングされたHTMLビュー

HTMLがレンダリングされたことが確認できました。図5.1で見た内容から、レスポンスのボディに以下のような文字列を返すことができると推論することができます。

```rust
HttpResponse::Ok()
    .content_type("text/html; charset=utf-8")
    .body("<h1>Items</h1>")
```

これは、文字列がHTML形式であれば、HTMLをレンダリングします。この啓示から、Rustサーバーが提供するHTMLファイルからHTMLをレンダリングするにはどうしたらいいと思いますか？先に進む前に、このことについて考えてみてください。これは、あなたの問題解決能力を鍛えることになるでしょう。

### ファイルから基本的なHTMLを読み取る

HTMLファイルがあれば、そのHTMLファイルを文字列に整えて、HttpResponseのボディにその文字列を挿入するだけで、レンダリングできる。そう、とても簡単なことなのです。これを実現するために、コンテンツローダーを構築します。

基本的なコンテンツローダーを構築するには、まずviews/app/content_loader.rsファイルにHTMLファイルの読み込み機能を構築します。

```rust
use std::fs;

pub fn read_file(file_path: &str) -> String {
    let data: String = fs::read_to_string(file_path).expect("Unable to read file");

    return data
}
```

ここでやらなければならないことは、レスポンスボディに必要なのはこれだけなので、文字列を返すことです。次に、views/app/mod.rsファイルの先頭にmod content_loader;という行を設けて、ローダーを定義します。

さて、ローディング機能ができたので、HTMLディレクトリが必要です。これは、srcディレクトリと一緒にtemplatesというディレクトリを定義することができます。templatesディレクトリの中に、templates/main.htmlというHTMLファイルを追加し、次のような内容を記述します。

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charSet="UTF-8"/>
        <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
        <meta httpEquiv="X-UA-Compatible" content="ie=edge"/>
        <meta name="description" content="This is a simple to do app"/>
        <title>To Do App</title>
    </head>
    <body>
        <h1>To Do Items</h1>
    </body>
</html>
```

ここでは、bodyタグに前回紹介した内容と同じ内容、つまり<h1>To Do Items</h1>があることがわかります。次に、metaタグの範囲を定義するheadタグがあります。viewportを定義していることがわかります。これは、ページコンテンツの寸法と拡大縮小をどのように扱うかをブラウザに指示するものです。アプリケーションは、さまざまなデバイスや画面サイズからアクセスされる可能性があるため、スケーリングは重要です。このビューポートで、ページの幅をデバイスの画面と同じ幅に設定することができます。そして、アクセスするページの初期スケールを1.0に設定することができます。httpEquivタグに移ると、X-UA-Compatibleに設定し、これは古いブラウザをサポートすることを意味します。最後のタグは、検索エンジンが使用できるページの説明文です。titleタグは、to doアプリがブラウザのタグに表示されることを保証するものです。これで、標準的なヘッダーのタイトルがボディに表示されるようになりました。

### ファイルから読み込んだ基本的なHTMLを提供する

さて、HTMLファイルを定義したので、それをロードして提供する必要があります。src/views/app/items.rsファイルに戻り、以下のコードでHTMLファイルをロードし、それを提供する必要があります。

```rust
use actix_web::HttpResponse;
use super::content_loader::read_file;

pub async fn items() -> HttpResponse {
    let html_data = read_file("./templates/main.html");
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(html_data)
}
```

アプリケーションを実行すると、次のような出力が得られます。


図5.2-HTMLページを読み込んだときの様子

図5.2では、以前と同じ出力になっていることがわかります。しかし、図5.2のタブに「To Do App」と表示されているのは、HTMLファイルのメタデータがビューに読み込まれていることを意味します。HTMLファイルを完全に利用することを妨げるものは何もありません。HTMLファイルが提供されるようになったので、次の野望であるページへの機能追加に移ることができます。

### HTMLファイルにJavaScriptを追加する

フロントエンドのユーザーにとって、ToDoアイテムの状態に対して何もできないのであれば、便利とは言えません。これを修正する前に、次の図を見て、HTMLファイルのレイアウトを理解する必要があります。


図5.3「HTMLファイルの一般的なレイアウト

ここで、図5.3では、ヘッダーにmetaタグを定義できることがわかる。しかし、ヘッダーの中にスタイル・タグを定義できることもわかります。ヘッダーの下のスタイルタグの中で、スタイルにCSSを挿入することができます。ボディーの下にはスクリプトセクションがあり、JavaScriptを挿入することができます。このJavaScriptはブラウザ上で実行され、ボディ内の要素と相互作用します。このように、CSSとJavaScriptを組み込んだHTMLファイルを提供することで、フロントエンドのシングルページアプリとして十分に機能することがわかります。これで、本章の導入部分を振り返ることができます。私はRustが大好きで、すべてをRustで書けと言いたい衝動に駆られますが、これはソフトウェア工学のどの言語にとっても良いアイデアではありません。JavaScriptで機能的なフロントエンドビューを簡単に提供できるようになった今、JavaScriptはフロントエンドのニーズに応える最良の選択と言えるでしょう。

### JavaScriptを使った当社サーバーとの通信

さて、HTMLファイルのどこにJavaScriptを挿入するかがわかったので、次はその機能をテストしてみましょう。このセクションの残りの部分では、HTMLボディにボタンを作成し、それをJavaScript関数に融合させ、そのボタンが押されたときにブラウザが入力されたメッセージのアラートを出力するようにします。これは、バックエンドアプリケーションには何の役にも立ちませんが、HTMLファイルについての理解が正しいことを証明するものです。次のコードを templates/main.html ファイルに追加します。

```html
<body>
    <h1>To Do Items</h1>
    <input type="text" id="name" placeholder="create to do item">
    <button id="create-button" value="Send">Create</button>
</body>
<script>
    let createButton = document.getElementById("create-button");
    createButton.addEventListener("click", postAlert);
    function postAlert() {
        let titleInput = document.getElementById("name");
        alert(titleInput.value);
        titleInput.value = null;
    }
</script>
```

bodyセクションで、inputとbuttonを定義していることがわかります。inputとbuttonのプロパティには、それぞれ固有のID名を付けています。そして、buttonのIDを使ってイベント・リスナーを追加しています。そして、ボタンがクリックされたときに実行されるpostAlert関数をそのイベントリスナーにバインドします。postAlert関数を起動すると、IDを使って入力を取得し、入力の値をアラートで出力します。そして、inputの値をnullに設定し、ユーザーが別の値を入力して処理できるようにします。新しいmain.htmlファイルに、inputにtestingを入れてボタンをクリックすると、次のような出力になります。


図5.4 「JavaScriptでアラートに接続した場合のボタンクリックの効果

JavaScriptは、bodyの中で要素が相互作用することにとどまる必要はありません。JavaScriptを使って、バックエンドのRustアプリケーションにAPIを呼び出すこともできます。しかし、急いでmain.htmlファイルにアプリケーション全体を書き込む前に、立ち止まって考える必要があります。そんなことをしたら、main.htmlファイルは巨大なファイルに膨れ上がってしまいます。デバッグが大変になります。また、コードが重複してしまう可能性もあります。同じJavaScriptを他のビューでも使いたい場合はどうすればいいのでしょうか。別のHTMLファイルにコピー＆ペーストしなければなりません。これでは拡張性が低く、関数を更新する必要がある場合、重複している関数の更新を忘れてしまう危険性があります。そこで、ReactのようなJavaScriptフレームワークが便利です。Reactについてはこの章の後半で説明しますが、今は、JavaScriptとHTMLファイルを分離する方法を考えることで、低依存なフロントエンドを完成させましょう。

このJavaScriptは、基本的にHTMLをその場で手動で書き換えていることに注意する必要があります。これを「やっつけ仕事」と表現する人もいるかもしれません。しかし、異なるアプローチの利点を真に理解するためには、Reactを探求する前に、私たちのアプローチを理解することが重要です。次のセクションに進む前に、src/views/to_do/create.rsファイルにあるcreateビューをリファクタリングする必要があります。これは、前の章で開発した内容を再確認する良い機会です。文字列ではなく、ToDoアイテムの現在の状態を返すように、作成ビューを基本的に変換する必要があります。これを試みると、ソリューションは次のようになるはずです。

```rust
use actix_web::HttpResponse;
use serde_json::Value;
use serde_json::Map;
use actix_web::HttpRequest;
use crate::to_do::{to_do_factory, enums::TaskStatus};
use crate::json_serialization::to_do_items::ToDoItems;
use crate::state::read_file;
use crate::processes::process_input;

pub async fn create(req: HttpRequest) -> HttpResponse {
    let state: Map<String, Value> = read_file("./state.json");
    let title: String = req.match_info().get("title").unwrap().to_string();
    let item = to_do_factory(&title.as_str(), TaskStatus::PENDING);
    process_input(item, "create".to_string(), &state);

    return HttpResponse::Ok().json(ToDoItems::get_state())
}
```

これで、すべてのToDo項目が最新になり、機能するようになりました。次のセクションに進み、フロントエンドがバックエンドに呼び出すようにします。
