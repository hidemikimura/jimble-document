<!-- https://jimble.io/ja/principles -->

# 考え方

jimble には守っている決めごとがいくつかあります。
機能を足すかどうかは、たいていここで決まります。

## 1. 上から順に追えること

コードを読む人が、`main` から目的の処理までを**指でたどれる**ことを最優先します。

そのために捨てたもの:

- **注釈でのルート定義**（`@GET @Path("/x")`）。
  どのクラスが読まれるのかがコードに出ないため
- **DI コンテナ**。
  `@Inject` された実装がどれになるかは、実行するまで分からないため
- **注釈でのトランザクション**（`@Transactional`）。
  どこで始まってどこで終わるかが、メソッドの外にあるため

かわりに `new` を書きます。`install(AdminController::new)` と書きます。
`try (DBTransaction transaction = ...)` と書きます。数行増えます。

## 2. 起動時にクラスパスを走査しない

コントローラもバッチも MCP のツールも、**自分で登録します**。

```java
tool("search_posts", SearchPostsTool::new);
tool("create_post", CreatePostTool::new);

resource("blog://latest", LatestPostsResource::new);
```

走査しないので、起動が速く、何が登録されたかが起動ログで完結し、
ネイティブイメージや jlink とも喧嘩しません。
登録し忘れは、起動ログのルート一覧で気づけます。

## 3. 黙って間違えないこと

いちばん高くつくバグは、落ちないバグです。
jimble は「静かに効かなくなる」状態を潰すことに時間を使っています。

例:

- `env=local` なのに `cookie.secure = true` なら、起動時に **WARN を出します**。
  そのままだとブラウザが Cookie を返さず、セッションも CSRF も Flash も
  エラーなしで効かなくなるためです
- トランザクションを開いたまま実行が終わったら、**ERROR を出してロールバックします**。
  黙って接続をプールへ返すと「入れたつもりが入っていない」が残ります
- ドキュメントのコード片は実コードから抜いています。
  印の付いた場所が消えたら、**サイトのビルドが落ちます**

## 4. 例外にしないところ

DB のエラーは例外ではなく戻り値で返します。
`select` 系は `null`、更新系は `-1` です。

```java
try (DB db = BlogExample.db()) {

	List<Data> rows = db.selectList(SQL.select().from(Post.instance()));

	/*
	 * DB のエラーは例外ではなく戻り値で返る（要件 F-D-11）。
	 * select 系は null、更新系は -1。
	 */
	if (rows == null) {
		Log.error("引けませんでした: " + db.getError());
		return;
	}

}
```

理由は、DB のエラーの多くが「呼び出し側が分岐したいもの」だからです。
例外にすると `try` で囲むか、囲み忘れて上まで飛ばすかの二択になります。

## 5. 一つの実行 = 一つの Context

Web のリクエスト・バッチの実行・MQ の1件・WebSocket の1メッセージ、
それぞれに `Context` が1つ作られます。
DB 接続もセッションもここにぶら下がり、終わりで畳まれます。

`Context` は `ScopedValue` で渡します。`ThreadLocal` ではありません。
仮想スレッドの上では `ScopedValue` のほうが安く、
かつ**書き換えられない**ので、途中ですり替わることがありません。

