## Reactアプリケーションをデスクトップアプリケーションへ変換する

Reactアプリケーションをデスクトップアプリケーションに変換することは複雑ではありません。そのためにElectronフレームワークを使用することになります。Electronは、JavaScript、HTML、CSSのアプリケーションを、macOS、Linux、Windowsの各プラットフォームでコンパイルされたデスクトップアプリケーションに変換してくれる強力なフレームワークです。Electronフレームワークは、暗号化されたストレージ、通知、電源モニター、メッセージポート、プロセス、シェル、システム設定など、APIを介してコンピュータのコンポーネントにアクセスすることも可能です。Slack、Visual Studio Code、Twitch、Microsoft Teamsなどのデスクトップアプリケーションは、Electronに組み込まれています。Reactアプリケーションを変換するには、package.jsonファイルを更新することから始めなければなりません。まず、package.jsonファイルの先頭にあるmetadataを以下のコードで更新する必要があります。

```json
{
  "name": "front_end",
  "version": "0.1.0",
  "private": true,
  "homepage": "./",
  "main": "public/electron.js",
  "description": "GUI Desktop Application for a simple To 
                  Do App",
  "author": "Maxwell Flitton",
  "build": {
    "appId": "Packt"
  },
  "dependencies": {
    . . .
```

そのほとんどは一般的なメタデータです。しかし、メインフィールドは必須です。ここに、Electronアプリケーションの実行方法を定義するファイルを記述します。また、homepageフィールドを"./"に設定することで、アセットのパスがindex.htmlファイルからの相対パスであることを保証します。メタデータが定義されたので、次の依存関係を追加します。

```json
"webpack": "4.28.3",
"cross-env": "^7.0.3",
"electron-is-dev": "^2.0.0"
```

これらの依存関係は、Electronアプリケーションの構築を支援します。これらが追加されたら、以下のコードでスクリプトを再定義することができます。

```json
    . . .
"scripts": {
    "react-start": "react-scripts start",
    "react-build": "react-scripts build",
    "react-test": "react-scripts test",
    "react-eject": "react-scripts eject",
    "electron-build": "electron-builder",
    "build": "npm run react-build && npm run electron-
              build",
    "start": "concurrently \"cross-env BROWSER=none npm run 
              react-start\" \"wait-on http://localhost:3000 
              && electron .\""
},
```

ここでは、Reactスクリプトの前にreactを付けています。これは、ReactのプロセスとElectronのプロセスを分離するためです。Reactアプリケーションをdevモードで動かすだけなら、以下のコマンドを実行する必要があります。

```bash
$ npm run react-start
```

また、Electronのビルドコマンドと開発開始コマンドを定義しました。Electronファイルを定義していないため、これらはまだ動作しません。package.jsonファイルの下部には、Electronアプリケーションをビルドするための開発者向け依存関係を定義する必要があります。

```json
    . . .
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  },
  "devDependencies": {
    "concurrently": "^7.1.0",
    "electron": "^18.0.1",
    "electron-builder": "^22.14.13",
    "wait-on": "^6.0.1"
  }
}
```

これで、package.json ファイルに必要なものをすべて定義しました。次のコマンドで、新しい依存関係をインストールする必要があります。

```bash
$ npm install
```

さて、ここで front_end/public/electron.js ファイルを作成し、Electron ファイルを作成できるようにしましょう。これは基本的にボイラープレートコードで、Electronでアプリケーションを動作させるための最小限のものなので、おそらく他のチュートリアルでもこのファイルを目にすることになるでしょう。まず、以下のコードで必要なものをインポートする必要があります。

```js
const { app, BrowserWindow } = require("electron");
const path = require("path");
const isDev = require("electron-is-dev");
```

次に、Desktopウィンドウを作成する関数を以下のコードで定義する必要があります。

```js
function createWindow() {
    const mainWindow = new BrowserWindow({
        width: 800,
        height: 600,
        webPreferences: {
            nodeIntegration: true,
            enableRemoteModule: true,
            contextIsolation: false,
        },
    });
    mainWindow.loadURL(
        isDev
           ? "http://localhost:3000"
           : `file://${path.join(__dirname, "../build/index.html")}`
    );
    if (isDev) {
        mainWindow.webContents.openDevTools();
    }
}
```

ここでは、基本的にウィンドウの幅と高さを定義しています。また、nodeIntegrationとenableRemoteModuleによって、レンダラーのリモートプロセス（ブラウザウィンドウ）がメインプロセス上でコードを実行できるようにしていることに注意してください。そして、メインウィンドウでURLの読み込みを開始します。開発者モードで実行した場合は、localhost上でReactアプリケーションを実行しているため、http://localhost:3000 を読み込むだけです。アプリケーションをビルドする場合は、私たちがコーディングしたアセットとファイルがコンパイルされ、./build/index.htmlファイルを通してロードすることができます。また、開発者モードで実行している場合は、開発者ツールを開くことを明記しています。以下のコードでウィンドウの準備ができたらcreateWindow関数を実行する必要があります。  

```js
app.whenReady().then(() => {
    createWindow();
    app.on("activate", function () {
        if (BrowserWindow.getAllWindows().length === 0){
           createWindow(); 
        }
    });
});
```

OSがmacOSの場合、ウィンドウを閉じても、プログラムを起動し続けなければなりません。

```js
app.on("window-all-closed", function () {
    if (process.platform !== "darwin") app.quit();
});
```

ここで、以下のコマンドを実行する必要があります。

```bash
$ npm start
```

これでElectronのアプリケーションが実行され、次のような出力が得られます。


図5.14 - Electronで動作するReactアプリケーション

図5.13では、アプリケーションがデスクトップ上のウィンドウで動作していることがわかります。また、画面上部にあるメニューバーからアプリケーションにアクセスできることもわかります。タスクバーには、アプリケーションのロゴが表示されています。


図5.15 - タスクバーのElectron

次のコマンドは、distフォルダにある私たちのアプリケーションをコンパイルし、クリックすると、アプリケーションをコンピュータにインストールします。

```bash
$ npm build
```

以下は、Macのアプリケーション領域で、私が作ったCamel for OasisLMFというオープンソースパッケージのGUIをElectronで試したときの例です。


図5.16-アプリケーション領域でのElectronアプリの紹介

最終的には、ロゴを考えることになります。しかし、これでブラウザでコンテンツを表示するこの章は終了です。
