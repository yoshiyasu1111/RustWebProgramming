## RustでWebアプリケーションをデザインする

前回は、Rustの構文について説明し、メモリ管理の癖を克服し、データ構造を構築することを可能にしました。しかし、経験豊富なエンジニアなら誰もが言うように、複数のファイルやディレクトリにまたがるコードを構造化することは、ソフトウェアを構築する上で重要な側面です。

この章では、基本的なコマンドラインToDoプログラムを構築します。コマンドラインプログラムの構築に必要な依存関係は、RustのCargoで管理します。このプログラムは、独自のモジュールを構築して管理し、それをプログラムの他の領域にインポートして利用するという、スケーラブルな方法で構成されます。私たちは、ToDoアプリケーションを作成、編集、削除する複数のファイルにまたがるToDoアプリケーションを構築することによって、これらの概念を学びます。このアプリケーションは、複数のToDoアプリケーションファイルをローカルに保存し、コマンドラインインターフェイスを使用してアプリケーションと対話することができます。

この章では、以下の項目について説明します。

- Cargoによるソフトウェアプロジェクトの管理
- コードの構造化
- 環境との相互作用

この章の終わりには、パッケージ化して使用できるアプリケーションをRustで構築できるようになります。また、自分のコードでサードパーティのパッケージを使用することもできるようになります。その結果、解決しようとする問題を理解し、論理的な塊に分解することができれば、サーバーやグラフィカルユーザーインターフェースを必要としない、あらゆるコマンドラインアプリケーションを構築することができるようになります。  