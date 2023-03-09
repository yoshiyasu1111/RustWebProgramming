## Reactのカスタムコンポーネントの作成

Appクラスを見てみると、HTMLをブラウザにレンダリングする方法やタイミングを管理するために、ステートとファンクションを活用できるクラスがあると便利であることがわかると思います。個々のToDo項目に関しては、ステートとファンクションを利用することができます。これは、ToDo項目から属性を取得し、Rustサーバを呼び出して編集または削除を行うボタンがあるためです。このセクションでは、src/components/ToDoItem.jsにToDoItemコンポーネントを、src/components/CreateToDoItem.jsにCreateToDoItemコンポーネントを作成することになります。アプリコンポーネントがアイテムのデータを取得し、アイテムをループして複数のToDoItemコンポーネントを作成するためです。そのために必要な手順がいくつかあるので、このセクションでは、以下のサブセクションに分割して説明します。

- ToDoItemコンポーネントを作成する
- CreateToDoItemコンポーネントを作成します。
- カスタムコンポーネントをAppコンポーネントで構築・管理する

さっそく始めてみましょう。

### ToDoItemコンポーネントを作成する

まずは、よりシンプルなToDoItemコンポーネントをsrc/components/ToDoItem.jsファイルに記述していきます。まず、以下のものをインポートする必要があります。

```js
import React, { Component } from 'react';
import axios from "axios";
```

これは何も新しいことではありません。必要なものをインポートできたので、次のコードでToDoItemをどのように定義するかに焦点を当てます。

```js
class ToDoItem extends Component {
    state = {
        "title": this.props.title,
        "status": this.props.status,
        "button": this.processStatus(this.props.status)
    }
    processStatus(status) {
        . . .
    }
    inverseStatus(status) {
        . . .
    }
    sendRequest = () => {
        . . .
    }
    render() {
        return(
            . . .
        )
    }
}
export default ToDoItem;
```

ここでは、コンポーネントの構築時にコンポーネントに渡されるパラメータであるthis.propsをステートに入力しています。次に、ToDoItemコンポーネントの関数を以下に示します。

- processStatusを使用する。PENDING などの ToDo 項目のステータスを、edit などのボタンに表示するメッセージに変換する機能。
- inverseStatus。ステータスがPENDINGのToDoアイテムがあり、それを編集した場合、そのステータスをDONEに変換して、ラストサーバーの編集エンドポイントに送信できるようにしたいのですが、その逆バージョンです。そこで、この関数では、渡されたステータスの逆数を作成します。
- sendRequestを実行します。この関数は、ToDoアイテムの編集または削除のためのリクエストをRustサーバに送信します。また、sendRequest関数がアロー関数であることもわかります。矢印構文では、関数をコンポーネントに結合して、render return文で関数を参照できるようにし、関数に結合されたボタンがクリックされたときにsendRequest関数を実行できるようにしています。

これで、関数の役割がわかったので、次のコードでステータス関数を定義することができます。

```js
processStatus(status) {
    if (status === "PENDING") {
        return "edit"
    } else {
        return "delete"
    }
}
inverseStatus(status) {
    if (status === "PENDING") {
        return "DONE"
    } else {
        return "PENDING"
    }
}
```

これは簡単なことなので、あまり説明は必要ありません。さて、ステータス処理関数ができたので、次のコードでsendRequest関数を定義することができます。

```js
sendRequest = () => {
    axios.post("http://127.0.0.1:8000/v1/item/" + this.state.button,
        {
            "title": this.state.title,
            "status": this.inverseStatus(this.state.status)
        },
    {headers: {"token": "some_token"}})
        .then(response => {
            this.props.passBackResponse(response);
        });
}
```

ここでは、押しているボタンによってエンドポイントが変わるので、this.state.buttonを使用してURLの一部を定義しています。また、this.props.passBackResponse関数を実行していることも確認できます。これは、ToDoItemコンポーネントに渡す関数です。これは、編集や削除のリクエストの後に、RustサーバからToDoアイテムの状態をすべて取得するためです。このため、Appコンポーネントを有効にして、受け渡されたデータを処理する必要があります。ここでは、「カスタムコンポーネントの構築と管理」サブセクションの「Appコンポーネント」で行うことを紹介します。App コンポーネントは、passBackResponse パラメーターの下に未実行の関数を持ち、ToDoItem コンポーネントに渡すことになります。この関数は、passBackResponseパラメータの下にあり、新しいToDoアイテムの状態を処理し、Appコンポーネントにレンダリングします。

これで、すべての関数の設定が完了しました。あとは、render関数の戻り値を定義するだけですが、以下のような形になります。

```html
<div>
    <p>{this.state.title}</p>
    <button onClick={this.sendRequest}>
                    {this.state.button}</button>
</div>
```

ここでは、ToDo項目のタイトルが段落タグでレンダリングされ、ボタンがクリックされるとsendRequest関数が実行されることが確認できます。これで、このコンポーネントは完成し、アプリケーションに表示する準備ができました。しかし、その前に、次のセクションでToDo項目を作成するコンポーネントを構築する必要があります。

### Reactでカスタムコンポーネントを作成する

私たちのReactアプリケーションは、ToDoアイテムの一覧表示、編集、削除については機能しています。しかし、ToDo項目を作成することはできません。これは、入力と作成ボタンで構成されており、ボタンをクリックすることで作成できるToDo項目を入れることができるようになっています。src/components/CreateToDoItem.jsでは、以下のものをインポートする必要があります。

```js
import React, { Component } from 'react';
import axios from "axios";
```

これらは、コンポーネントを構築するための標準的なインポートです。インポートが定義されると、CreateToDoItemコンポーネントは次のような形になります。

```js
class CreateToDoItem extends Component {
    state = {
        title: ""
    }
    createItem = () => {
        . . .
    }
    handleTitleChange = (e) => {
        . . .
    }
    render() {
        return (
            . . .
        )
    }
}
export default CreateToDoItem;
```

先のコードで、CreateToDoItemコンポーネントが以下の機能を持つことがわかります。

- createItem: Rustサーバーにリクエストを送信し、タイトルを持つToDoアイテムを作成する状態です。
- handleTitleChangeを使用します。この関数は、入力が更新されるたびに状態を更新する

この2つの関数を調べる前に、これらの関数をコーディングする順序を反転させ、次のコードでrender関数のリターンを定義します。

```html
<div className="inputContainer">
    <input type="text" id="name"
           placeholder="create to do item"
           value={this.state.title}
           onChange={this.handleTitleChange}/>
    <div className="actionButton"
         id="create-button"
         onClick={this.createItem}>Create</div>
</div>
```

ここでは、入力の値がthis.state.titleであることを確認しています。また、入力が変化したときには、this.handleTitleChange関数を実行しています。さて、ここまででrender関数について説明しましたが、新たに紹介することはありません。この機会に、もう一度CreateToDoItemコンポーネントの概要を見て、createItem関数とhandleTitleChange関数を自分で定義してみてください。これらは、ToDoItemコンポーネントの関数と同じような形をしています。

createItem関数とhandleTitleChange関数を定義しようとすると、以下のような感じになります。

```js
createItem = () => {
    axios.post(
        "http://127.0.0.1:8000/v1/item/create/" + this.state.title,
        {},
        {headers: {"token": "some_token"}})
            .then(response => {
                this.setState({"title": ""});
                this.props.passBackResponse(response);
        });
}
handleTitleChange = (e) => {
    this.setState({"title": e.target.value});
}    
```

これで、両方のカスタム・コンポーネントの定義が完了しました。次のサブセクションでは、カスタム・コンポーネントを管理する準備が整いました。  

### カスタムコンポーネントをAppコンポーネントで構築・管理する

カスタムコンポーネントを作成するのは楽しいですが、アプリケーションで使わなけれ ばあまり意味がありません。このサブセクションでは、src/App.js ファイルにいくつかの追加コードを追加して、カスタムコンポーネントを使用できるようにすることにします。まず、以下のコードでコンポーネントをインポートする必要があります。

```js
import ToDoItem from "./components/ToDoItem";
import CreateToDoItem from "./components/CreateToDoItem";
```

さて、コンポーネントができたので、最初の改造に移ります。AppコンポーネントのprocessItemValues関数は、以下のコードで定義することができます。

```js
processItemValues(items) {
    let itemList = [];
    items.forEach((item, _) => {
        itemList.push(
            <ToDoItem key={item.title + item.status}
                title={item.title}
                status={item.status.status}
                passBackResponse={this.handleReturnedState}
            />
        )
    })
    return itemList
}
```

ここでは、Rustサーバから取得したデータをループ処理していますが、一般的なHTMLタグにデータを渡すのではなく、ToDo項目データのパラメータを独自のカスタムコンポーネントに渡し、HTMLタグのように扱っていることがわかります。返された状態で独自のレスポンスを処理するとなると、以下のコードでデータを処理し、状態を設定する矢印関数であることがわかります。

```js
handleReturnedState = (response) => {
  let pending_items = response.data["pending_items"]
  let done_items = response.data["done_items"]
  this.setState({
      "pending_items": this.processItemValues(pending_items),
      "done_items": this.processItemValues(done_items),
      "pending_items_count": response.data["pending_item_count"],
      "done_items_count": response.data["done_item_count"]
  })
}
```

これは、getItems関数と非常によく似ています。重複するコードを減らしたいのであれば、ここでリファクタリングを行うこともできます。ただし、これを実現するためには、render関数のreturn文を次のコードで定義する必要があります。

```html
<div className="App">
    <h1>Pending Items</h1>
    <p>done item count: 
    {this.state.pending_items_count}</p>
    {this.state.pending_items}
    <h1>Done Items</h1>
    <p>done item count: {this.state.done_items_count}</p>
    {this.state.done_items}
    <CreateToDoItem 
     passBackResponse={this.handleReturnedState} />
</div>
```

ここでは、createItemコンポーネントを追加した以外には、あまり変化がないことがわかります。RustサーバーとReactアプリケーションを実行すると、次のようなビューが表示されます。


図5.12「カスタムコンポーネントを使用したReactアプリケーションのビュー

図5.12は、カスタムコンポーネントがレンダリングされていることを示しています。ボタンをクリックすると、すべてのAPIコールが動作し、カスタムコンポーネントが正常に動作することが確認できます。あとは、フロントエンドを見栄えよくするために、CSSをReactアプリケーションに取り込むだけです。
