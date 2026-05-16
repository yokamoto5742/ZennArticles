# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

Zenn の記事・本を管理するリポジトリ。`zenn-cli` を使用して記事の作成・プレビューを行う。

## よく使うコマンド

```bash
# 記事の新規作成
npx zenn new:article

# 本の新規作成
npx zenn new:book

# プレビューサーバー起動（http://localhost:8000）
npx zenn preview
```

## ディレクトリ構成

- `articles/` — Zenn 記事ファイル（`.md`）
- `books/` — Zenn 本のディレクトリ
- `.claude/rules/` — Claude Code 向けルール定義

## Zenn Markdown 記法

記事は Zenn 独自の Markdown 拡張に対応している。主な要素：

- メッセージボックス: `:::message` / `:::message alert`
- アコーディオン: `:::details タイトル`
- リンクカード: URL のみの行
- コードブロック: `` ```言語:ファイル名 ``
- 数式: KaTeX 形式

詳細は `.claude/rules/Zenn_Markdown_Reference.md` を参照。
