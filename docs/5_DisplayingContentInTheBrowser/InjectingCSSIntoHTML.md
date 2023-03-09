## HTMLにCSSを組み込む

CSSの注入は、JavaScriptの注入と同じアプローチをとります。HTMLファイル内にCSSタグを用意し、そのCSSをファイル内のCSSに置き換えるのです。これを実現するために、以下の手順を実行する必要があります。

1. HTMLファイルにCSSタグを追加する。
2. アプリ全体のベースとなるCSSファイルを作成します。
3. メインビューのCSSファイルを作成します。
4. CSSとJavaScriptを提供するためにRust crateを更新します。
5. このプロセスを詳しく見てみましょう。

### HTMLにCSSタグを追加する

まず、templates/main.html ファイルを変更します。

```html
 <style>
    {{BASE_CSS}}
    {{CSS}}
</style>
<body>
    <div class="mainContainer">
        <h1>Done Items</h1>
        <div id="doneItems"></div>
        <h1>To Do Items</h1>
        <div id="pendingItems"></div>
        <div class="inputContainer">
            <input type="text" id="name"
                   placeholder="create to do item">
            <div class="actionButton" 
                 id="create-button" 
                 value="Send">Create</div>
        </div>
    </div>
</body>
<script>
    {{JAVASCRIPT}}
</script>
```

ここでは、2つのCSSタグがあることが確認できます。BASE_CSS}}タグは、画面サイズに応じた背景色や列の比率など、複数の異なるビューで統一されるベースCSSを記述するためのタグです。BASE_CSS}}タグは、このビューのCSSクラスを管理するためのものです。敬称略、css/base.cssとcss/main.cssは、私たちのビューのために作られたファイルです。また、mainContainerというクラスを持つdivの中に全てのアイテムを入れていることに注目してください。これにより、すべてのアイテムを画面上で中央に配置することができます。さらに、CSSで参照できるようにクラスを追加し、アイテムの作成ボタンをボタンのHTMLタグからdivのHTMLタグに変更しました。これが終わると、javascript/main.jsファイルのrenderItems関数が、アイテムのループのために次のように変更されます。

```js
function renderItems(items, processType, 
    elementId, processFunction) {
    . . . 
    for (i = 0; i < items.length; i++) {
        . . .
        placeholder += '<div class="itemContainer">' +
            '<p>' + title + '</p>' +
            '<div class="actionButton" ' + 
                  'id="' + placeholderId + '">'
            + processType + '</div>' + "</div>";
        itemsMeta.push({"id": placeholderId, "title":        title});
    }
    . . .
}
```

これを考慮して、css/base.cssファイルにベースCSSを定義することができるようになりました。

### ベースとなるCSSを作成する

次に、ページとその構成要素のスタイルを定義する必要があります。まず、css/base.cssファイルにページのボディを定義することから始めるとよいでしょう。次のコードで、ボディの基本的な設定を行うことができます。

```css
body {
    background-color: #92a8d1;
    font-family: Arial, Helvetica, sans-serif;
    height: 100vh;
} 
```

背景色は、色の種類を示すリファレンスです。この参照は、見ただけでは意味がないように思えるかもしれませんが、オンラインではカラーピッカーがあり、色を見て選ぶと参照コードが提供されます。コードエディターによってはこの機能をサポートしているものもありますが、手っ取り早く参考にしたい場合は、HTMLカラーピッカーでググれば、無料で利用できるオンラインのインタラクティブなツールがたくさん出てきます。先ほどの設定で、ページ全体の背景は#92a8d1というネイビーブルーのコードになります。これだけだと、ページの大半の背景が白になってしまいます。紺色の背景は、コンテンツがあるところだけ存在することになります。高さは100vhに設定しました。vhは、ビューポートの高さの1%に相当します。このことから、100vhは、bodyで定義したスタイリングがビューポートの100％を占めることを意味すると推測できる。次に、Arial、Helvetica、sans-serifに上書きされない限り、すべてのテキストのフォントを定義する。font-familyで複数のフォントを定義していることがわかります。これは、そのすべてが実装されているわけでも、ヘッダーやHTMLタグのレベルに応じて異なるフォントが用意されているわけでもありません。その代わり、これはフォールバックメカニズムです。まず、ブラウザはArialのレンダリングを試みます。ブラウザがサポートしていない場合は、Helveticaのレンダリングを試み、それも失敗した場合は、sans-serifのレンダリングを試みます。

これで、本体の一般的なスタイルを定義しましたが、異なるスクリーンサイズについてはどうでしょうか？例えば、携帯電話でアプリケーションにアクセスする場合は、異なるサイズのボディを用意する必要があります。次の図を見てください。


図5.7 「スマホとデスクトップモニターの余白の違い

図5.7は、ToDo項目のリストが変化したときに、余白とスペースが占める割合を示したものである。携帯電話の場合、画面占有率が低いので、画面の大半をToDo項目で占めなければならない。しかし、ワイドスクリーンのデスクトップモニターを使えば、画面のほとんどをToDo項目が占める必要はありません。もし比率が同じなら、ToDo項目はX軸に引き伸ばされ、読みづらく、率直に言って見栄えも悪くなってしまいます。そこで登場するのがメディアクエリです。ウィンドウの幅や高さなどの属性によって、異なるスタイル条件を設定することができるのです。まずはスマホの仕様から。つまり、画面の幅が500ピクセルまでなら、css/base.cssファイルの中で、bodyに以下のCSS構成を定義する必要があります。

```css
@media(max-width: 500px) {
    body {
        padding: 1px;
        display: grid;
        grid-template-columns: 1fr;
    }
}
```

ここでは、ページの端と各要素の周りのパディングがちょうど1ピクセルであることがわかります。また、グリッド表示もあります。ここでは、列と行を定義することができます。しかし、ここではその機能をフルに活用していません。1列だけです。つまり、図5.7のスマホの描写にあるように、ToDo項目が画面の大半を占めることになります。この文脈ではグリッドを使用していませんが、大画面用の他の構成との関係がわかるように、グリッドを入れておきました。画面がもう少し大きくなれば、ページを3つの縦列に分けることができますが、真ん中の列の幅と両側の列の幅の比率は5：1になります。これは、画面がまだあまり大きくないため、表示するアイテムが画面の大部分を占めるようにしたいからです。このような場合は、別のパラメータを持つメディアクエリを追加して調整することができます。

```css
@media(min-width: 501px) and (max-width: 550px) {
    body {
        padding: 1px;
        display: grid;
        grid-template-columns: 1fr 5fr 1fr;
    } 
    .mainContainer {
        grid-column-start: 2;
    }
}
```

また、ToDo項目を配置するmainContainer CSSクラスでは、grid-column-start属性を上書きしていることがわかります。このようにしないと、mainContainerは幅1frで左の余白に押し込まれることになります。その代わり、5frの真ん中で始まり、終わることになります。grid-column-finish属性で、mainContainerを複数の列にまたがるようにすることができます。  

画面が大きくなった場合、アイテムの幅が制御不能にならないように、比率をさらに調整したい。そのためには、中央のカラムと両サイドのカラムの比率を3対1にし、画面の幅が1001pxを超えたら1対1にする必要があります。

```css
@media(min-width: 551px) and (max-width: 1000px) {
    body {
        padding: 1px;
        display: grid;
        grid-template-columns: 1fr 3fr 1fr;
    } 
    .mainContainer {
        grid-column-start: 2;
    }
} 
@media(min-width: 1001px) {
    body {
        padding: 1px;
        display: grid;
        grid-template-columns: 1fr 1fr 1fr;
    } 
    .mainContainer {
        grid-column-start: 2;
    }
}
```

これで、すべてのビューに対して一般的なCSSを定義できたので、次にビュー固有のCSSをcss/main.cssファイルに記述することができるようになりました。

### トップページのCSSを作成する

さて、アプリのコンポーネントを分解する必要があります。ToDoのリストがあります。リストの各項目は、異なる背景色を持つdivになります。

```css
.itemContainer {
    background: #034f84;
    margin: 0.3rem;
}
```

このクラスはマージンが0.3であることがわかります。remを使用しているのは、マージンをルート要素のフォントサイズに比例して拡大縮小させたいからです。また、カーソルがアイテムの上に乗ったときに、色がわずかに変化するようにしたい。

```css
.itemContainer:hover {
    background: #034f99;
}
```

アイテムコンテナ内では、アイテムのタイトルを段落タグで表記しています。アイテムコンテナ内のすべての段落のスタイルを定義したいが、それ以外の場所では定義しない。コンテナ内の段落のスタイルを定義するには、次のコードを使用します。

```css
.itemContainer p {
    color: white;
    display: inline-block;
    margin: 0.5rem;
    margin-right: 0.4rem;
    margin-left: 0.4rem;
}
```

inline-blockは、タイトルをdivと一緒に表示させることができます。marginの定義は、タイトルがアイテムコンテナの端にぴったりとくるのを防ぐだけです。また、段落の色が白であることを確認しています。

アイテムのタイトルがスタイリングされたので、残るアイテムのスタイリングは、編集か削除のどちらかのアクションボタンだけとなります。このアクションボタンは、クリックする場所がわかるように、背景色を変えて右側に表示させます。そのためには、次のコードのように、ボタンのスタイルをクラスで定義する必要があります。

```css
.actionButton {
    display: inline-block;
    float: right;
    background: #f7786b;
    border: none;
    padding: 0.5rem;
    padding-left: 2rem;
    padding-right: 2rem;
    color: white;
}
```

ここでは、displayを定義し、右にフロートするようにし、背景色とpaddingを定義しています。これで、以下のコードを実行することで、ホバー時に色が変わるようにすることができます。

```css
.actionButton:hover {
    background: #f7686b;
    color: black;
}
```

さて、ここまでですべての概念を網羅したので、次は入力コンテナのスタイルを定義する必要があります。これは、次のコードを実行することで行うことができます。

```css
.inputContainer {
    background: #034f84;
    margin: 0.3rem;
    margin-top: 2rem;
}
.inputContainer input {
    display: inline-block;
    margin: 0.4rem;
}
```

完成しましたすべてのCSS、JavaScript、HTMLを定義しました。アプリを実行する前に、メインビューにデータをロードする必要があります。

### RustからCSSとJavaScriptを提供する

CSSは、views/app/items.rsファイルで提供されます。HTML、JavaScript、ベースCSS、メインCSSの各ファイルを読み込みます。そして、HTMLデータのタグを、他のファイルのデータに置き換えます。  

```rust
pub async fn items() -> HttpResponse {
    let mut html_data = read_file("./templates/main.html");
    let javascript_data: String = read_file("./javascript/main.js");
    let css_data: String = read_file("./css/main.css");
    let base_css_data: String = read_file("./css/base.css");
    html_data = html_data.replace("{{JAVASCRIPT}}", &javascript_data);
    html_data = html_data.replace("{{CSS}}", &css_data);
    html_data = html_data.replace("{{BASE_CSS}}", &base_css_data);
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(html_data)
}
```

これで、サーバーをスピンアップすると、次のスクリーンショットのような直感的なフロントエンドを持つアプリが完全に動作するようになります。


図5.8 「CSS後のメインページ

アプリが機能し、ベースとなるCSSとHTMLが設定されていても、独自のCSSを持つ再利用可能なスタンドアロンHTML構造を持ちたいと思うことがあります。これらの構造体は、必要に応じてビューに注入することができます。このように、コンポーネントを一度書いておけば、他のHTMLファイルにインポートすることができます。これにより、メンテナンスが容易になり、複数のビューでコンポーネントの一貫性を確保することができます。例えば、ビューの上部にインフォメーションバーを作成した場合、他のビューでも同じスタイルで表示させたいと思うでしょう。したがって、次のセクションで説明するように、情報バーをコンポーネントとして一度作成し、それを他のビューに挿入することは理にかなっています。
