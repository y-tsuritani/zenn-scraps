# Zenn CLI

## 概要

このリポジトリは Zenn の記事投稿・管理するためのリポジトリです。

- [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## 使い方

**新しい記事の作成**

```bash
`$ npx zenn new:article`
```

**ヘッダーのテンプレート**

```yaml
---
title: "" # 記事のタイトル
emoji: "⚙" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["Python", "GCP"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---
```

**記事のプレビュー**

```bash
$ npx zenn preview # プレビュー開始
```

**公開時刻の予約**

```yaml
published: true # trueを指定する
published_at: 2050-06-12 09:03 # 未来の日時を指定する
```
