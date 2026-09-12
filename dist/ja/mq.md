<!-- https://jimble.io/ja/mq -->

# MQ

jimble の MQ は **DB のテーブルをキューにします**。
別のミドルウェアは要りません。トランザクションを共有できるのが大きいところです。

## 書く

```java
public class NoticeExecutor extends MqExecutor {

	@Override
	public String queueName () { return "mq_blog"; }

	@Override
	public String key () { return "notice"; }

	@Override
	public MqExecuteType executeType () { return MqExecuteType.short_time; }

	@Override
	public MqStatus execute (DB db, Data row) {

		Data data = row.getDataOptional("data");

		// ... 処理する

		return MqStatus.completed;

	}

}
```

- `queueName()` はテーブル名です。キューごとにテーブルを分けます
- `key()` は種別です。1つのテーブルに複数の種類を混ぜられます
- `executeType()` は `short_time` / `long_time`。取り出し方が変わります
- 戻り値が `MqStatus.completed` 以外なら、`maxRetry()` まで再実行されます

## 積む

```java
try (DBTransaction transaction = new DBTransaction(db)) {

	transaction.beginTransaction();

	long id = db.insert(
		SQL.insert(Post.instance())
			.value(Post.title, request.getString("title"))
			.value(Post.body, request.getString("body"))
			.value(Post.image_name, request.getStringOptional("image_name"))
			.value(Post.published, request.getBoolean("published"))
			.value(Post.created_at, new Date())
	);

	if (id <= 0) {
		transaction.rollbackEndTransaction();
		return -1;
	}

	new NoticeExecutor().put(db, new Data()
		.putData("post_id", id)
		.putData("title", request.getString("title")));

	transaction.commitEndTransaction();

	/*
	 * コミットしてから流す。
	 * 先に流すと、ロールバックしたときに
	 * 「入っていない記事のお知らせ」だけが届く。
	 */
	PostFeedHandler.notifyNewPost(request.getStringOptional("title"));

	return id;

}
```

`put()` は**トランザクションの中で呼びます**。
同じ DB なので、ロールバックすれば積んだものも消えます。
「メールだけ飛んで注文が入っていない」が起きません。

時間を指定して積むこともできます。

```java
new NoticeExecutor().put(db, data, scheduledAt);
```

## 二重に実行される

**同じメッセージが2回実行されることがあります。**
処理の途中でプロセスが落ちたときなどです。

なので、**何度やっても同じになるように書いてください**。

- メールなら、送信済みの記録を見てから送る
- 集計なら、足すのではなく入れ直す
- 外部 API なら、冪等キーを付ける

「ちょうど1回」は分散システムでは作れません。jimble はそのふりをしません。

## 登録して回す

```java
// テーブルを用意する
new MqQueue(NoticeExecutor.QUEUE_NAME).install();

// Executor を登録する（クラスパス走査はしない）
MqRegistry.add(NoticeExecutor::new);
```

取り出して回すのはバッチです。`MqWorkerBatch` のようなバッチを1つ作り、
[バッチ](./batch) と同じように登録して cron で回します。

