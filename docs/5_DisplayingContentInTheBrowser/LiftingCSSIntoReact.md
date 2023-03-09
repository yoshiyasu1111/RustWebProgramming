## CSSをReactに持ち込む

Reactアプリケーションを使えるようにするための最後の一歩を踏み出しました。CSSを複数の異なるファイルに分割することも可能です。しかし、この章も終盤に差し掛かり、もう一度CSSを見直すと、不必要に重複したコードでこの章を埋め尽くしてしまいます。HTMLとJavaScriptは異なりますが、CSSは同じものです。そこで、以下のファイルからすべてのCSSをコピーして実行できるようにします。

- テンプレート/コンポーネント/ヘッダー.css
- css/ベース.css
- css/main.css

ここに記載されているCSSファイルをfront_end/src/App.cssファイルにコピーします。CSSには1つだけ変更があり、以下のコードスニペットに示すように、すべての.body参照を.Appに置き換える必要があるところです。

```css
.App {
  background-color: #92a8d1;
  font-family: Arial, Helvetica, sans-serif;
  height: 100vh;
}
@media(min-width: 501px) and (max-width: 550px) {
  .App {
    padding: 1px;
    display: grid;
    grid-template-columns: 1fr 5fr 1fr;
  }
  .mainContainer {
    grid-column-start: 2;
  }
}
. . .
```

これで、CSSをインポートして、アプリやコンポーネントで使用できるようになりました。また、レンダー関数の戻り値のHTMLを変更する必要があります。3つのファイルすべてを通して作業することができます。src/App.jsファイルでは、次のコードでCSSをインポートする必要があります。

```js
import "./App.css";
```

次に、ヘッダーを追加し、divタグを正しいクラスで定義し、render関数の戻り文に次のコードを指定します。

```html
<div className="App">
    <div className="mainContainer">
        <div className="header">
            <p>complete tasks: 
            {this.state.done_items_count}</p>
            <p>pending tasks: 
            {this.state.pending_items_count}</p>
        </div>
        <h1>Pending Items</h1>
        {this.state.pending_items}
        <h1>Done Items</h1>
        {this.state.done_items}
        <CreateToDoItem passBackResponse=
       {this.handleReturnedState}/>
    </div>
</div>
```

src/components/ToDoItem.jsファイルでは、次のコードでCSSをインポートする必要があります。

```js
import "../App.css";
```

次に、ボタンをdivに変更し、render関数のreturn文を以下のコードで定義する必要があります。

```html
<div className="itemContainer">
    <p>{this.state.title}</p>
    <div className="actionButton" onClick={this.sendRequest}>
        {this.state.button}
    </div>
</div>
```

次に、ボタンをdivに変更し、render関数のreturn文を以下のコードで定義する必要があります。

```html
<div className="inputContainer">
    <input type="text" id="name"
           placeholder="create to do item"
           value={this.state.title}
           onChange={this.handleTitleChange}/>
    <div className="actionButton"
         id="create-button"
         onClick={this.createItem}>
            Create
    </div>
</div>
```

これで、RustのWebサーバからReactアプリケーションにCSSを持ち込むことができました。RustサーバーとReactアプリケーションを実行すると、次の図のような出力が得られます。


図5.13「CSSを追加したReactアプリケーションのビュー

そして、これだ!Reactアプリケーションは動作しています。Reactアプリケーションを立ち上げて実行するまでに時間がかかりますが、Reactの方が柔軟性があることがわかります。また、文字列を手動で操作する必要がないため、Reactアプリケーションでエラーが発生しにくいことも確認できます。さらに、Reactで構築する利点はもう1つあり、それは既存のインフラストラクチャです。次回は、ReactアプリケーションをElectronでラッピングすることで、Reactアプリケーションをコンピュータのアプリケーションで動作するコンパイル済みデスクトップアプリケーションに変換する予定です。
