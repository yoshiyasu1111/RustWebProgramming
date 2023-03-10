## HTTPリクエストの処理

ここまでは、基本的なビューを提供するためにActix Webフレームワークを利用してきました。しかし、リクエストからデータを抽出し、ユーザーにデータを返すとなると、これだけでは限界があります。本章では、第2章「RustでWebアプリケーションを設計する」と第3章「HTTPリクエストを処理する」のコードを融合し、ToDo項目を処理するサーバビューを構築します。そして、ビューをより使いやすくするために、データを抽出して返すためのJSONシリアライズを探求します。また、ビューに表示される前にミドルウェアでヘッダーからデータを抽出します。ToDoアプリケーションの作成、編集、削除のエンドポイントを構築することで、データのシリアライズとリクエストからのデータの抽出に関する概念を探求します。

この章では、以下の項目について説明します。

- コードを定着させるための初期設定を知る
- ビューにパラメータを渡す
- JSONシリアライズのためのマクロの使用
- ビューからデータを抽出する

この章を終えると、URL、JSONを使ったボディ、HTTPリクエストのヘッダーでデータを送受信できる基本的なRustサーバーを構築することができるようになります。これは基本的に、データを保存するための適切なデータベース、ユーザーを認証する機能、ブラウザにコンテンツを表示する機能を持たない、完全に機能するAPI Rustサーバーです。しかし、これらのコンセプトは次の3つの章でカバーされます。あなたは、完全に動作するRustサーバを稼働させるためのホームランを踏んでいます。さっそく始めてみましょう
