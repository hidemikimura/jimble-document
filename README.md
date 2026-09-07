# jimble-document

<https://jimble.io> の中身。**Cloudflare Workers（静的アセット）がこのリポジトリを配っている。**

```
wrangler.jsonc   配り方の設定（手で管理する）
README.md        これ（手で管理する）
dist/            サイト本体。すべて生成物
```

**資産は `dist/` だけ**である。ルートに置いたものは配られない。
根拠：`wrangler.jsonc` を資産のディレクトリに入れていた頃は
`https://jimble.io/wrangler.jsonc` が 200 で返っていた。

## dist/ を直さないこと

**すべて生成物である。** 原稿は本体（<https://github.com/hidemikimura/jimble>）の
`docs/site/ja/*.md` にあり、コード片は `examples/` とテストの実コードから抜いている。

`dist/` で直しても、次に生成したときに消える。

## 更新のしかた

本体のリポジトリで：

```bash
./gradlew :jimble-docs:site
```

`docs/site/build/` に出たものを、このリポジトリの `dist/` へ丸ごと置き換えて push する。

```bash
rm -rf dist && mkdir dist
cp -r <本体>/docs/site/build/. dist/
git add -A && git commit -m "ドキュメント更新" && git push
```

`wrangler.jsonc` と `README.md` は `dist/` の外なので、置き換えても消えない。

## wrangler.jsonc

```jsonc
{
	"name": "<ダッシュボードの Worker 名>",
	"compatibility_date": "2026-09-07",
	"assets": {
		"directory": "./dist",
		"not_found_handling": "404-page",
		"html_handling": "auto-trailing-slash"
	}
}
```

| 設定 | なぜ要るか |
|---|---|
| `not_found_handling: "404-page"` | **Workers は既定でこれをしない。**無いと、見つからないパスが<b>本文が空の 404</b> になる（従来の Pages は自動で `404.html` を返すが、Workers は返さない） |
| `html_handling: "auto-trailing-slash"` | `/ja/routing.html` を `/ja/routing` へ正規化する。既定と同じ値だが、設定ファイルを置いた以上は明示しておく |

## リンクに拡張子を付けていない

ファイルは `dist/ja/routing.html` だが、リンクは `/ja/routing` である。
`.html` を付けると **307 で正規化されて1回よけいに往復する**ので、
本文のリンクも sitemap も検索の遷移先も拡張子なしで出している。

**そのため、ファイルを直接ブラウザで開くとリンクが辿れない。**
手元で見るときはサーバー越しにする。

```bash
cd dist && python3 -m http.server 8080
```

## dist/ の中身

| ファイル | 役目 |
|---|---|
| `index.html` | `/` に来た人を `/ja/` へ送る（`_redirects` が効かないときの保険） |
| `_redirects` | `/` → `/ja/` の 302 |
| `_headers` | CSP ほかの安全側のヘッダ。`*.workers.dev` は検索避け |
| `sitemap.xml` / `robots.txt` | |
| `404.html` | |
| `ja/*.html` | 本文 20 ページ |
| `ja/search-index.json` | 検索の索引（素の JavaScript が読む） |
| `static/` | `site.css` と `search.js`。**外部の CDN には繋いでいない** |

これらも生成物である。手で足すと、生成し直すたびに消える。
足すなら本体の `jimble-docs/src/main/java/io/jimble/docs/DocsBuilder.java` に足す。
