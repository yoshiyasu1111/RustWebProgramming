## PostgreSQLによるデータ永続化

この本のこの時点では、アプリケーションのフロントエンドは定義されており、私たちのアプリは額面通りに動作しています。しかし、私たちのアプリはJSONファイルから読み書きを行っていることが分かっています。

本章では、JSONファイルを廃止し、データを保存するためのPostgreSQLデータベースを導入します。そのために、Dockerを使ったデータベース開発環境を構築します。また、Dockerデータベースコンテナを監視する方法についても調査します。次に、データベースのスキーマを構築するためにマイグレーションを作成し、データベースとやり取りするためのデータモデルをRustで構築します。そして、作成、編集、削除のエンドポイントがJSONファイルの代わりにデータベースとやり取りするように、アプリをリファクタリングします。

この章では、以下の項目について説明します。

- PostgreSQLデータベースの構築
- DieselでPostgreSQLに接続する。
- アプリケーションをPostgreSQLに接続する
- アプリケーションを構成する
- データベース接続プールの管理

この章の終わりには、PostgreSQLデータベースのデータの読み書きや削除を行うアプリケーションを、データモデルで管理できるようになります。データモデルに変更を加えた場合は、マイグレーションで管理することができるようになります。これができれば、プールでデータベース接続を最適化し、データベース接続ができない場合はビューにヒットする前にサーバーがHTTPリクエストを拒否するようにすることができるようになります。
