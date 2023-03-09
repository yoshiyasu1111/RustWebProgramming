## コンポーネントの継承

時には、ビューに注入できるコンポーネントを作りたいと思うことがあります。そのためには、CSSとHTMLの両方を読み込んで、HTMLの正しい部分に挿入する必要があります。

そのためには、コンポーネント名を受け取り、コンポーネント名からタグを作成し、コンポーネント名に基づいてHTMLとCSSをロードするadd_component関数を作成します。この関数は、views/app/content_loader.rs ファイルに定義することにします。

```rust
pub fn add_component(component_tag: String, 
    html_data: String) -> String {
    let css_tag: String = component_tag.to_uppercase() + "_CSS";
    let html_tag: String = component_tag.to_uppercase() + "_HTML";
    let css_path = String::from("./templates/components/") + &component_tag.to_lowercase() + ".css";
    let css_loaded = read_file(&css_path);
    let html_path = String::from("./templates/components/") + &component_tag.to_lowercase() + ".html";
    let html_loaded = read_file(&html_path);
    let html_data = html_data.replace(html_tag.as_str(), &html_loaded);
    let html_data = html_data.replace(css_tag.as_str(), &css_loaded);
    return html_data
} 
```

ここでは、同ファイルで定義されているread_file関数を使用します。そして、コンポーネントのHTMLとCSSをビューデータにインジェクションします。templates/components/ディレクトリにコンポーネントを入れ子にしていることに注意してください。今回は、ヘッダーコンポーネントを挿入するので、add_component関数にヘッダーを渡すと、header.htmlとheader.cssファイルを読み込もうとします。templates/components/header.html ファイルでは、以下の HTML を定義する必要があります。

```html
<div class="header">
    <p>complete tasks: </p><p id="completeNum"></p>
    <p>pending tasks: </p><p id="pendingNum"></p>
</div>
```

ここでは、完了したToDoと保留中のToDoの件数を表示するだけです。templates/components/header.css ファイルで、次の CSS を定義する必要があります。

```css
.header {
    background: #034f84;
    margin-bottom: 0.3rem;
}
.header p {
    color: white;
    display: inline-block;
    margin: 0.5rem;
    margin-right: 0.4rem;
    margin-left: 0.4rem;
}
```

add_component関数がCSSとHTMLを正しい位置に挿入するためには、templates/main.htmlファイルの<style>セクションにHEADERタグを挿入する必要があります。

```html
. . . 
    <style>
        {{BASE_CSS}}
        {{CSS}}
        HEADER_CSS
    </style>
    <body>
        <div class="mainContainer">
            HEADER_HTML
            <h1>Done Items</h1>
. . .
```

HTMLとCSSがすべて定義できたので、views/app/items.rsファイルにadd_component関数をインポートする必要があります。

```rust
use super::content_loader::add_component;
```

同じファイル内で、items view関数内に、次のようにヘッダーを追加する必要があります。

```rust
html_data = add_component(String::from("header"), html_data);
```

ここで、ヘッダーがToDo項目のカウントで更新されるように、injecting_header/javascript/main.jsファイルのapiCall関数を変更する必要があります。

```rust
document.getElementById("completeNum").innerHTML = JSON.parse(this.responseText)["done_item_count"];
document.getElementById("pendingNum").innerHTML = JSON.parse(this.responseText)["pending_item_count"]; 
```

コンポーネントを挿入すると、次のようなレンダリングビューが得られます。


図5.9 「ヘッダーのあるメインページ

見ての通り、ヘッダーはデータを正しく表示しています。ビューのHTMLファイルにヘッダタグを追加し、ビューでadd_componentを呼び出すと、そのヘッダを取得することができます。

今現在、私たちは完全に動作するシングルページのアプリケーションを手に入れました。しかし、これは困難がなかったわけではありません。フロントエンドに機能を追加し始めると、フロントエンドが制御不能に陥り始めることが目に見えています。そこで、Reactのようなフレームワークの出番です。Reactを使えば、コードを適切なコンポーネントに構造化して、必要なときにいつでも使えるようにすることができるのです。次のセクションでは、基本的なReactアプリケーションを作成します。
