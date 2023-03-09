## フロントエンドに認証を追加する

ログイン機能を組み込みます。まず、src/components/LoginForm.js ファイルにログインフォームを作成することから始めます。まず、以下のものをインポートします。

```js
import React, {Component} from 'react';
import axios from 'axios';
import '../css/LoginForm.css';
```

インポートしたCSSのコードは、本章の付録で提供しています。繰り返しのコードが多いので、ここでは割愛します。また、GitHubのレポからCSSのコードをダウンロードすることもできます。これらのインポートにより、以下のコードでログインフォームのフレームワークを構築することができます。

```js
class LoginForm extends Component {
    state = {
        username: "",
        password: "",
    }
    submitLogin = (e) => {
        . . .
    }
    handlePasswordChange = (e) => {
        this.setState({password: e.target.value})
    }
    handleUsernameChange = (e) => {
        this.setState({username: e.target.value})
    }
    render() {
        . . .
    }
}
export default LoginForm;
```

ここでは、ユーザー名とパスワードを記録しており、常に状態を更新していることがわかります。状態が更新されると、render関数が実行されることを思い出してください。これは強力な機能で、好きなように変更することができます。例えば、ユーザー名の長さがある一定の長さを超えたら、コンポーネントの色を変更したり、ボタンを削除したりすることができます。この本の範囲外なので、私たち自身が劇的な変更をすることはありません。さて、フレームワークの定義ができたので、次のコードでレンダー関数が何を返すかを明記します。

```html
<form className="login" onSubmit={this.submitLogin}>
    <h1 className="login-title">Login</h1>
    <input type="text" className="login-input"
    placeholder="Username"
    autoFocus onChange={this.handleUsernameChange}
           value={this.state.username} />
    <input type="password" className="login-input"
    placeholder="Password"
    onChange={this.handlePasswordChange}
           value={this.state.password} />
    <input type="submit" value="Lets Go"
    className="login-button" />
</form>
```

ここでは、フォームにユーザー名とパスワードのフィールドがあり、変更があったときにhandleUsernameChange関数とhandlePasswordChange関数を実行していることがわかります。ユーザー名とパスワードを入力したら、submitLogin関数でこれらのフィールドをバックエンドに送信する必要がありますが、これはここで定義できます。

```rust
submitLogin = (e) => {
    e.preventDefault();
    axios.post("http://localhost:8000/v1/auth/login",
        {"username": this.state.username,
         "password": this.state.password},
        {headers: {"Access-Control-Allow-Origin": "*"}}
        )
        .then(response => {
            this.setState({username: "", password: ""});
            this.props.handleLogin(response.data["token"]);
        })
        .catch(error => {
            alert(error);
            this.setState({password: "", firstName: ""});
        });
}
```

ここでは、ログインAPI呼び出しのレスポンスを、propsを使って通した関数に渡していることがわかります。これはsrc/app.jsファイル内で定義する必要があります。エラーが発生した場合は、アラートでこれを出力して、何が起こったかを教えてくれます。いずれにせよ、ユーザー名とパスワードのフィールドを空にします。

ログインフォームを定義したところで、ユーザーがログインする必要があるときに、このフォームを表示する必要があります。ユーザーがログインした後は、ログインフォームを非表示にする必要があります。その前に、src/app.jsファイルに以下のコードでログインフォームをインポートしておく必要があります。

```js
import LoginForm from "./components/LoginForm";
```

ここで、ログイン状態を追跡する必要があります。これを実現するために、Appクラスのstateは以下のような形にする必要があります。

```js
state = {
  "pending_items": [],
  "done_items": [],
  "pending_items_count": 0,
  "done_items_count": 0,
  "login_status": false,
}
```

項目を管理していますが、login_statusがfalseの場合、ログインフォームを表示することができます。ユーザーがログインしたら、login_statusをtrueに設定し、その結果、ログインフォームを非表示にすることができます。ログイン状態を記録するようになったので、AppクラスのgetItems関数を更新することができます。

```js
getItems() {
  axios.get("http://127.0.0.1:8000/v1/item/get",
  {headers: {"token": localStorage.getItem("user-token")}})
  .then(response => {
      let pending_items = response.data["pending_items"]
      let done_items = response.data["done_items"]
      this.setState({
        "pending_items":
         this.processItemValues(pending_items),
        "done_items": this.processItemValues(done_items),
        "pending_items_count":
         response.data["pending_item_count"],
        "done_items_count":
         response.data["done_item_count"]
        })
  }).catch(error => {
      if (error.response.status === 401) {
        this.logout();
      }
  });
}
```

トークンを取得し、ヘッダーに載せていることがわかります。不正なコードでエラーが発生した場合は、Appクラスのlogout関数を実行します。このログアウト関数は、ここで定義された形式をとります。

```js
logout() {
  localStorage.removeItem("token");
  this.setState({"login_status": false});
}
```

ローカルストレージからトークンを削除し、login_statusをfalseに設定していることがわかります。このログアウト関数は、ToDoアイテムの編集時にエラーが発生した場合にも実行される必要があります。トークンは期限切れになる可能性があるため、どこででも発生する可能性があり、再ログインを促さなければならないことを覚えておく必要があります。つまり、以下のコードでToDoItemコンポーネントにlogout関数を渡す必要があります。

```js
processItemValues(items) {
  let itemList = [];
  items.forEach((item, _)=>{
      itemList.push(
          <ToDoItem key={item.title + item.status}
                    title={item.title}
                    status={item.status}
                    passBackResponse={this.handleReturnedState}
                    logout={this.logout}/>
      )
  })
  return itemList
}
```

ToDoItemコンポーネントにログアウト関数を渡したら、src/components/ToDoItem.jsファイルのToDoアイテムを編集するAPIコールを次のコードで更新します。

```js
sendRequest = () => {
    axios.post("http://127.0.0.1:8000/v1/item/" +
                this.state.button,
        {
            "title": this.state.title,
            "status": this.inverseStatus(this.state.status)
        },
    {headers: {"token": localStorage.getItem(
         "user-token")}})
        .then(response => {
            this.props.passBackResponse(response);
        }).catch(error => {
            if (error.response.status === 401) {
                this.props.logout();
            }
    });
}
```

ここでは、ローカルストレージからAPI呼び出しにヘッダーを介してトークンを渡していることがわかります。そして、不正なステータスを取得した場合には、propsを介して渡されたlogout関数を実行します。

ここで、src/app.jsファイルに戻り、アプリケーションの機能をまとめました。このアプリケーションは、最初にアクセスするときにデータを読み込む必要があることを忘れないでください。アプリケーションの初期読み込み時に、次のコードでローカルストレージのトークンを考慮する必要があります。

```js
componentDidMount() {
  let token = localStorage.getItem("user-token");
  if (token !== null) {
      this.setState({login_status: true});
      this.getItems();
  }
}
```

これで、このアプリケーションはトークンがあるときだけバックエンドからアイテムを取得するようになります。レンダリング機能でアプリケーションをまとめる前に、ログインだけを処理する必要があります。トークンをローカルストレージで処理する方法はおわかりいただけたと思います。この時点で、AppクラスのhandleLogin関数を自分で構築することができるはずです。もし、自分で関数を作ってみたなら、次のようなコードになるはずです。

```js
handleLogin = (token) => {
  localStorage.setItem("user-token", token);
  this.setState({"login_status": true});
  this.getItems();
}
```

現在、Appクラスのレンダー関数を定義する段階に来ています。ログイン状態がtrueの場合、次のコードでアプリケーションのすべてを表示することができます。

```js
if (this.state.login_status === true) {
    return (
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
                <CreateToDoItem
                 passBackResponse={this.handleReturnedState}/>
            </div>
        </div>
    )
}
```

ここでは、あまり新しいことはありません。しかし、ログイン状態がtrueでない場合は、次のコードでログインフォームを表示すればよい。

```js
else {
    return (
        <div className="App">
            <div className="mainContainer">
                <LoginForm handleLogin={this.handleLogin}
                />
            </div>
        </div>
    )
}
```

見ての通り、LoginFormコンポーネントにhandleLogin関数を渡しています。これで、アプリケーションを実行する準備ができました。最初のビューは次のような感じです。


図7.8 - ログインして、アプリケーションのローディングビューを表示します。

正しい認証情報を入力すると、アプリケーションにアクセスし、ToDo項目を操作することができるようになります。このアプリケーションは、基本的に動作しています。
