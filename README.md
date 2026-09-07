# jimble-document

<https://jimble.io> の中身。**Cloudflare Pages がこのリポジトリをそのまま配っている。**

## ここを直さないこと

**すべて生成物である。** 原稿は本体（<https://github.com/hidemikimura/jimble>）の
`docs/site/ja/*.md` にあり、コード片は `examples/` とテストの実コードから抜いている。

ここで直しても、次に生成したときに消える。

## 更新のしかた

本体のリポジトリで：

```bash
./gradlew :jimble-docs:site
```

`docs/site/build/` に出たものを、このリポジトリの直下へ丸ごと上書きして push する。
Cloudflare Pages が push を拾って配り直す。

## Cloudflare Pages の設定

| 項目 | 値 |
|---|---|
| ビルドコマンド | （なし） |
| 出力ディレクトリ | `/` |
| ルートディレクトリ | `/` |

ビルドは本体側で済んでいるので、Pages 側では何もしない。

## 生成物に含まれるもの

| ファイル | 役目 |
|---|---|
| `index.html` | `/` に来た人を `/ja/` へ送る（`_redirects` が効かないときの保険） |
| `_redirects` | `/` → `/ja/` の 302 |
| `_headers` | CSP ほかの安全側のヘッダ。`*.pages.dev` は検索避け |
| `sitemap.xml` / `robots.txt` | |
| `404.html` | |
| `ja/*.html` | 本文 20 ページ |
| `ja/search-index.json` | 検索の索引（素の JavaScript が読む） |
| `static/` | `site.css` と `search.js`。**外部の CDN には繋いでいない** |

これらも生成物である。手で足すと、生成し直すたびに消える。
足すなら本体の `jimble-docs/src/main/java/io/jimble/docs/DocsBuilder.java` に足す。
