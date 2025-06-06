---
slug: welcome
title: Welcome
authors: [slorber, yangshun]
tags: [facebook, hello, docusaurus]
---

[Docusaurusのブログ機能](https://docusaurus.io/docs/blog)は[blogプラグイン](https://docusaurus.io/docs/api/plugins/@docusaurus/plugin-content-blog)によって提供されています。

`blog`ディレクトリにMarkdownファイル（またはフォルダ）を追加するだけで利用できます。

定期的にブログを投稿する著者は`authors.yml`に追加できます。

ブログ投稿の日付は以下のようなファイル名から自動的に抽出されます：

- `2019-05-30-welcome.md`
- `2019-05-30-welcome/index.md`

ブログ投稿用フォルダを使用すると、関連画像を一緒に管理するのに便利です：

![Docusaurusのぬいぐるみ](./docusaurus-plushie-banner.jpeg)

ブログではタグ機能もサポートされています！

**ブログ機能が不要な場合**：このディレクトリを削除し、Docusaurus設定で`blog: false`を指定してください。