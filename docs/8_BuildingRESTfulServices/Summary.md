## まとめ

この章では、RESTful 設計のさまざまな側面を理解し、それをアプリケーションに実装しました。アプリケーションのレイヤーを評価し、ミドルウェアをリファクタリングすることで、結果に応じて2つの異なるフューチャーを処理できるようにしました。これは、単にリクエストを承認するだけにとどまりません。リクエストのパラメータに基づいて、他のサーバーにリクエストをリダイレクトするミドルウェアを実装したり、フロントエンドに何らかの変更を加えてから別のAPIコールを行うコードオンデマンドレスポンスで直接応答したりすることができるのです。このアプローチでは、ビューがヒットする前にミドルウェアで複数の未来の結果を持つカスタムロジックという別のツールを手に入れることができます。

そして、パス構造体をリファクタリングしてインターフェースを統一し、フロントエンドとバックエンドのビューが衝突するのを防ぎました。

そして、ロギングのさまざまなレベルを検討し、すべてのリクエストをログに記録して、静かでありながら望ましくない動作を明らかにしました。フロントエンドをリファクタリングしてこれを修正した後、過剰なAPIコールを防ぐためにToDoアイテムをフロントエンドにキャッシュする際に、ロギングを使ってキャッシュ機構が正しく機能しているかどうかを評価しました。これで、私たちのアプリケーションは合格点です。しかし、このアプリケーションをサーバーにデプロイして、監視したり、何か問題が起きたときにログをチェックしたり、ToDoリストを持つ複数のユーザーを管理したり、不正なリクエストをビューに到達する前に拒否したりできるような段階には至っていません。また、キャッシュも備えており、アプリケーションはステートレスで、PostgreSQLとRedisのデータベースにデータをアクセスしたり書き込んだりしています。

次章では、Rust構造体のユニットテストとAPIエンドポイントの機能テストを記述し、デプロイに向けたコードのクリーニングを行います。
