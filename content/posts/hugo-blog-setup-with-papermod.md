---
date: '2026-02-03T09:13:49+09:00'
draft: true
title: 'HugoとPaperModでブログを構築する'
categories: ["Tech"]
tags: ["Hugo", "PaperMod", "ブログ", "静的サイト"]
summary: "静的サイトジェネレーターHugoとPaperModテーマを使って、シンプルで高速なブログを構築する方法を解説します。"
---

## はじめに

ブログを始めるにあたり、WordPressなどのCMSを使う方法もありますが、今回は静的サイトジェネレーターの**Hugo**を選びました。理由は以下の通りです。

- 表示速度が非常に速い
- セキュリティリスクが低い
- Markdownで記事が書ける
- GitHubでバージョン管理できる
- ホスティングが無料（GitHub Pages, Cloudflare Pagesなど）

## Hugoのインストール

### macOS（Homebrew）

```bash
brew install hugo
```

### Windows（Chocolatey）

```bash
choco install hugo-extended
```

インストール後、バージョンを確認します。

```bash
hugo version
```

## プロジェクトの作成

```bash
hugo new site my-blog
cd my-blog
```

## PaperModテーマの導入

PaperModは、シンプルで高速、かつ機能が充実したテーマです。Git Submoduleとして追加します。

```bash
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

## 設定ファイルの作成

`hugo.yaml`を作成し、基本設定を記述します。

```yaml
baseURL: https://example.com/
languageCode: ja
title: My Blog
theme: ["PaperMod"]

params:
  defaultTheme: auto
  ShowReadingTime: true
  ShowCodeCopyButtons: true
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
```

## 記事の作成

```bash
hugo new posts/my-first-post.md
```

作成されたファイルを編集します。

```markdown
---
date: '2026-02-03'
draft: false
title: '初めての記事'
categories: ["Tech"]
tags: ["Hugo"]
---

ここに本文を書きます。
```

## ローカルでプレビュー

```bash
hugo server -D
```

ブラウザで `http://localhost:1313` にアクセスすると、サイトを確認できます。`-D`オプションをつけると、`draft: true`の記事もプレビューできます。

## ビルドとデプロイ

本番用にビルドするには、以下を実行します。

```bash
hugo
```

`public/`ディレクトリに静的ファイルが生成されます。これをGitHub PagesやCloudflare Pagesにデプロイすれば公開完了です。

## まとめ

HugoとPaperModを使えば、シンプルで高速なブログを簡単に構築できます。Markdownで記事を書けるので、エンジニアにとっては非常に快適な執筆環境です。

次回は、カスタマイズやデプロイ自動化について書く予定です。
