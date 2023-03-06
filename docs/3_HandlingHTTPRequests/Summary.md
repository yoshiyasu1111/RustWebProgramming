## まとめ

この章では、スレッド、フューチャー、非同期関数の基本を説明しました。その結果、マルチサーバーソリューションを実際に見て、何が起こっているのかを確信を持って理解することができました。これを踏まえて、前章で学んだ概念を基に、ビューを定義するモジュールを構築しました。さらに、ビューをその場で構築してサーバーに追加できるように、ファクトリーを連鎖させるようにしました。この連鎖するファクトリーの仕組みにより、サーバーの構築時にビューモジュール全体を構成から出し入れすることができるようになりました。

また、パスを定義するユーティリティ構造体を構築し、ビューのセットに対するURLの定義を標準化しました。今後の章では、このアプローチを使って、認証、JSONシリアライズ、フロントエンドモジュールを構築していく予定です。ここまでの内容で、次の章では、さまざまな方法でユーザーからデータを抽出して返すビューを構築することができるようになります。このようにモジュール化することで、ロジックが分離され設定可能で、コードを管理しやすく追加できるRustで、実世界のWebプロジェクトを構築するための強力な基盤を手に入れることができます。

次の章では、リクエストとレスポンスの処理に取り組みます。params、body、header、formをビューに渡し、JSONを返すことで処理する方法を学びます。この新しいメソッドを、前章で作成したToDoモジュールと組み合わせて使用し、サーバービューを通じてToDoアイテムとのやり取りを可能にします。