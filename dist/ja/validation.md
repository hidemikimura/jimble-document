<!-- https://jimble.io/ja/validation -->

# 検証とページング

## 検証は3段ある

| 段 | クラス | 何をするか |
| --- | --- | --- |
| 1項目 | `ValidationRule` | 「空でない」「1〜120 の整数」を積む |
| 1リクエスト | `ValidationRules` | 列ごとにルールを結びつけ、まとめて回す |
| 1ルート | `ValidationExecutor` | 失敗したら**後続の処理を止めて 422 を返す** |

下だけ、上だけ、どちらでも使えます。

## ルールを組み立てる

```java
ValidationRule rule = new ValidationRule()
	.empty()
	.textLengthMax(100);
```

積んだ順に走り、**最初に落ちたところで止まります**（1項目につきエラーは1件）。

| 分類 | メソッド |
| --- | --- |
| 必須 | `empty()` / `required()`（同じもの） |
| 文字数 | `textLength(min, max)` / `textLengthMin(min)` / `textLengthMax(max)` |
| バイト数 | `textByteLength(min, max[, charset])` / `textByteLengthMin` / `textByteLengthMax`（既定 UTF-8） |
| 数値 | `integer()` / `integer(min, max)` / `integerMin` / `integerMax` / `number()` / `number(min, max)` / `numberMin` / `numberMax` |
| 形式 | `bool()` / `email()` / `url()` / `domain()` / `date()` / `date(format)` / `regex(regex)` / `enumType(Class)` |
| 文字種 | `characterType(CharacterType[])` / `characterType(Character[])` / 両方 |
| 自作 | `custom(IValidator)` |
| 分岐 | `insertRequired()` |

> [!NOTE]
> **`empty()` 以外は、空を通します。**
> `textLengthMax(100)` は「入っているなら 100 文字以内」という意味で、
> 空文字や `null` はエラーにしません。**必須は必ず `empty()` で書いてください。**

> [!WARN]
> `regex(...)` は `matches` ではなく **`find`（部分一致）** です。
> 全体に効かせたいときは `^` と `$` を自分で書いてください。

## 列にまとめる

```java
ValidationRules rules = new ValidationRules()
	.put(Item.name, new ValidationRule().empty())
	.put(Item.age, new ValidationRule().integer(1, 120));

Data request = new Data();
request.putData(Item.name, "");
request.putData(Item.age, "999");

// エラーは最初の1件で止めず、全部集める（要件 F-V-03）
Data errors = rules.validate(null, request);

Data messages = ValidationMessages.toMessages(errors);
```

- **項目をまたいだエラーは全部集めます**（1件ずつ言われるのが、入力し直す人にはいちばん困るため）
- **送られてこなかった項目は検証しません**（`insertRequired()` を除く）
- 1つの列に配列が来たら、要素ごとに回します
- `put(rule)`（列なし）で、項目に紐づかない相関チェックも書けます

### 生のエラーの形

`validate` が返すのは**文言ではありません。**「どの種別で落ちたか」と「そのときの設定」です。

```java
{ "validation_type": Empty, "validation_setting": {}, "input": "" }
```

これを人が読む形に変えるのが `ValidationMessages.toMessages(errors)` で、
**項目名 → メッセージの一覧**になります。

```json
{ "title": ["入力してください"], "age": ["1 以上 120 以下の整数で入力してください"] }
```

> [!TIP]
> 文言を持たずに種別で返しているので、**同じ検証結果を日本語にも英語にも API のコードにも変えられます。**

### 文言を差し替える

```java
ValidationMessages.put(ValidationErrorType.Empty, (type, settings) -> "required");

ValidationRules rules = new ValidationRules()
	.put(Item.name, new ValidationRule().empty());

Data request = new Data();
request.putData(Item.name, "");

assertEquals(List.of("required"),
	ValidationMessages.toMessages(rules.validate(null, request)).get("name"));
```

> [!TRAP]
> `ValidationMessages` は **static でグローバル**です。テストで差し替えたら
> `finally` で `ValidationMessages.reset()` してください。
> 忘れると、**あとから走ったテストだけが落ちます。**

## 登録のときだけ必須

```java
ValidationRules rules = new ValidationRules()
	.put(Item.name, new ValidationRule().insertRequired().empty())
	.insertRequestChecker(req -> req.getBoolean("is_insert"));

Data update = new Data();
update.put("is_insert", false);
assertTrue(rules.validate(null, update).isEmpty(), "更新なのに必須になっている");

Data insert = new Data();
insert.put("is_insert", true);
assertFalse(rules.validate(null, insert).isEmpty(), "登録なのに必須になっていない");
```

「登録リクエストかどうか」の判定は `insertRequestChecker` に渡します。
同じ判定が各バリデータの `isInsertRequest` にも届きます。

## 複数行を検証する

```java
ValidationRules rules = new ValidationRules()
	.put(Item.name, new ValidationRule().empty());

Data ok = new Data();
ok.putData(Item.name, "あ");

Data ng = new Data();
ng.putData(Item.name, "");

List<Data> errors = rules.validate(null, List.of(ok, ng, ok));

assertEquals(1, errors.size());
assertEquals(2, errors.getFirst().getInt("index"), "行番号が違う");
```

**エラーのある行だけ**返り、各行に `index` が入ります（**1 始まり**）。

## ルートにかける

`ValidationExecutor` を継承して `validate` だけ書きます。

```java
private class Failing extends ValidationExecutor {

	@Override
	protected void validate (WebContext context) {

		log.add("validate");
		addError("title", "入力してください");

	}

}
```

ルートでは**いちばん先に積みます。**

```java
JimbleApp app = new JimbleApp() {
	{
		post("/items", context -> {
			context.addExecutor(new Failing());
			context.addExecutor(new UseCase());
		});
	}
};
```

失敗すると、こうなります。

| | |
| --- | --- |
| ステータス | **422** |
| 本文 | `{"validation": {"項目名": ["メッセージ"]}}` **＋ 送られてきた入力値** |
| 後続の Executor | **実行されない**（捨てられる） |
| `error(...)` フック | **通らない**（例外を投げていないため。[エラー処理](./errors)） |

`ValidationRules` の結果をそのまま積むこともできます。

```java
addErrors(rules.validate(db, context.request().bodyAll()));
```

> [!NOTE]
> `ValidationExecutor` は **`WebContext` をフィールドに持ちません。**
> `validate(WebContext)` の引数で受け取ります。
> インスタンスを使い回したときに、**前のリクエストのコンテキストに書き込む**事故を避けるためです。

> [!WARN]
> **列定義からルールは自動生成されません。**
> 生成されたテーブルクラス（`Post.title` など）が持っているのは
> 型・NULL 可否・主キーだけで、**varchar の桁数を持っていません。**
> `textLengthMax` を自動で導く材料が無いので、ルールは手で書きます。

## ページング

### リクエストから読む

```java
Paging paging = context.request().paging();
```

| | |
| --- | --- |
| 読むキー | `page` / `per` |
| 既定 | `page = 1`、`per = 10` |
| 全件 | `per=all`（LIMIT を付けません） |
| キー名の変更 | `paging.page` / `paging.per`（設定） |
| 数値でない値 | 無視して既定を使います |

`context.request().paging(20)` と書けば、`per` が来ていないときの既定を変えられます。

### SELECT にかける

```java
Paging paging = new Paging();
paging.load(request("2", "10"), 0);

SelectListResponse response = DBUtil.getMainDB().selectListWithRowCount(select().paging(paging));

assertEquals(10, response.list.size(), "1ページ分だけ取れていない");
assertEquals(TOTAL, response.rowCount, "総件数が LIMIT に影響されている");

assertEquals(TOTAL, paging.totalCount());
assertEquals(3, paging.maxPage(), "25 件を 10 件ずつなら 3 ページ");
assertEquals(11, paging.start());
```

`selectListWithRowCount` が**総件数の COUNT も投げます。**
COUNT 文は FROM / WHERE / GROUP BY / HAVING だけを写すので、
SELECT 句や ORDER BY、LIMIT には影響されません。

> [!TRAP]
> **総件数を数えるのは `selectListWithRowCount` だけ**です。
> ふつうの `selectList(builder)` では `paging.totalCount()` が 0 のままで、
> **ページャが「1 / 1 ページ」になります。**

> [!WARN]
> 生 SQL 版（`selectListWithRowCount(sql, params...)`）は `Paging` を触りません。
> 自分で `paging.set(response.list.size(), response.rowCount)` を呼んでください。

### 取れるもの

| メソッド | 中身 |
| --- | --- |
| `page()` | 現在ページ |
| `per()` | 1ページの件数 |
| `perAll()` | 全件指定だったか |
| `totalCount()` | 総件数 |
| `maxPage()` | 総ページ数（**最低 1**） |
| `start()` | このページの先頭が何件目か（**1 始まり**） |
| `count()` | このページで実際に取れた件数 |

> [!NOTE]
> **「次がある / 前がある」のメソッドはありません。**
> `page() > 1` と `page() < maxPage()` で判定してください。

### レスポンスに載る

`context.request().paging()` を**呼んだ時点で**レスポンスに載ります。自分で詰め直す必要はありません。

```json
{
  "rows": [ ... ],
  "paging": { "page": 2, "per": 10, "perAll": false,
              "maxPage": 3, "totalCount": 25, "start": 11, "count": 10 }
}
```

テンプレートからも同じものが見えます（`${paging.page()}`）。

### 気をつけること

> [!TRAP]
> **`per` に上限がありません。**`?per=100000` を送られると
> `LIMIT 100000` がそのまま出ます。外に公開する一覧では、
> `paging(...)` の前後で自分で上限を決めてください。

範囲外のページ（`?page=999`）はエラーにならず、**0 件が返ります。**

