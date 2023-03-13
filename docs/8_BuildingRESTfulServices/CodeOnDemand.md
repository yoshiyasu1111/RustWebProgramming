## コードオンデマンド

コードオンデマンドは、バックエンドサーバーがフロントエンドのコードを直接実行するものです。この制約はオプションであり、広く使われているわけではありません。しかし、バックエンドサーバに、フロントエンドでコードを実行するときとそのタイミングを決定する権利を与えるので、便利です。ログアウトビューでは、フロントエンドでJavaScriptを直接実行し、それを文字列で返しています。これは、src/views/auth/logout.rs ファイルで行われています。ローカルストレージにToDoアイテムを追加したことを忘れてはいけません。もし、ログアウト時にこれらのアイテムをローカルストレージから削除しなければ、他の誰かが2分以内に同じコンピュータの自分のアカウントにログインした場合、私たちのToDoアイテムにアクセスできてしまうでしょう。このような可能性は極めて低いですが、念には念を入れた方がよいでしょう。src/views/auth/logout.rsファイルにあるlogoutビューは、次のような形をしていることを思い出してください。

```rust
pub async fn logout() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(. . .)
}
```

レスポンスのボディ内には、次のような内容があります。

```rust
"<html>\
<script>\
    localStorage.removeItem('user-token'); \
    localStorage.removeItem('item-cache-date'); \
    localStorage.removeItem('item-cache-data-pending'); \
    localStorage.removeItem('item-cache-data-done'); \
    window.location.replace(
        document.location.origin);\
</script>\
</html>"
```

これにより、ユーザートークンを削除しただけでなく、すべての項目と日付も削除しました。これで、ログアウトした後もデータは安全です。
