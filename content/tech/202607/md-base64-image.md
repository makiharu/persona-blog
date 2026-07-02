---
date: '2026-07-02T10:00:00+09:00'
draft: false
title: 'Markdownに画像をbase64で直接埋め込む'
categories: ["Tech"]
tags: ["Markdown", "base64", "tips"]
summary: "外部ファイルなしでMarkdownに画像を埋め込む方法のメモ。Data URIスキームを使う。"
---

## やりたいこと

・Shapointでmarkdownファイルをプレビュー表示できるようにする  
・今後、格納先のファイルパスは変わる可能性があり、ファイルパスの変更は面倒なので、base64化して、mdに画像を埋め込むことで、画像の参照を楽にしたい。

```markdown
![alt](./image.png)
```

これをファイルなしで、Markdown1ファイルだけで画像を表示させたい。

## そもそもbase64とは

バイナリデータ（画像・動画など）をテキストとして表現するためのエンコード方式。

画像ファイルの中身は人間には読めないバイナリデータ（0と1の羅列）だが、MarkdownやHTMLはテキスト形式なのでバイナリをそのまま書けない。そこで **base64** を使い、バイナリをテキストに変換する。

使う文字は以下の64種類。大文字・小文字は別々に数える。

| 種類 | 文字 | 数 |
|---|---|---|
| 大文字 | A〜Z | 26 |
| 小文字 | a〜z | 26 |
| 数字 | 0〜9 | 10 |
| 記号 | `+` `/` | 2 |
| **合計** | | **64** |

この64文字だけで構成されるテキストに変換する。名前の由来は「64種類の文字を使う」から。

```
image.png の中身（バイナリ）
  ↓ base64エンコード
iVBORw0KGgoAAAANSUhEUgAAAAUA...（テキスト）
  ↓ Markdown に直接貼り付けられる
```

デコードすれば元のバイナリに戻せる。名前の由来は「64種類の文字を使う」から。

## 方法: Data URIスキーム

画像をbase64エンコードしてData URIとして直接埋め込む。

```markdown
![alt](data:image/png;base64,<base64文字列>)
```

フォーマットは `data:<MIMEタイプ>;base64,<データ>` 。

## base64変換コマンド

### macOS / Linux

```bash
base64 -i image.png | tr -d '\n'
```

`base64` コマンドはデフォルトで76文字ごとに改行を入れて出力する。Data URI に改行が混ざると URL として壊れた扱いになり画像が表示されないため、`tr -d '\n'` で改行をすべて除去して1行にする。

- `|` — 左のコマンドの出力を右のコマンドに渡す（パイプ）
- `tr -d '\n'` — `tr` は文字を変換・削除するコマンド。`-d` は削除モード、`'\n'` は改行文字

### Python

```python
import base64

with open("image.png", "rb") as f:
    encoded = base64.b64encode(f.read()).decode()

print(f"data:image/png;base64,{encoded}")
```

## 埋め込み例

```markdown
![ロゴ](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA...)
```

## メリット・デメリット

| | |
|---|---|
| メリット | ファイル1つで完結する、画像リンク切れが起きない |
| デメリット | ファイルサイズが大きくなる（元サイズの約1.33倍）、diff が読みにくくなる |

## 使いどころ

- GitHubのREADMEで画像パスの管理が面倒なとき
- ドキュメントを1ファイルで配布したいとき
- メールやチャットにMarkdownをそのままペーストするとき

逆に、頻繁に変更する画像や大きい画像は素直に外部ファイルにした方がよい。

## 理解度チェック

### 問1

base64とは何をするための仕組みか？

<details>
<summary>答え</summary>
バイナリデータをテキストとして表現するためのエンコード方式。画像などのバイナリをMarkdownやHTMLに直接書けるテキストに変換できる。
</details>

### 問2

base64が「64」と呼ばれる理由は？使う文字の内訳も答えよ。

<details>
<summary>答え</summary>
64種類の文字を使うから。内訳は大文字A〜Z（26）＋小文字a〜z（26）＋数字0〜9（10）＋記号「+」「/」（2）＝64。
</details>

### 問3

Markdownに画像をbase64で埋め込むときの書き方は？

<details>
<summary>答え</summary>

```markdown
![alt](data:image/png;base64,<base64文字列>)
```

フォーマットは `data:<MIMEタイプ>;base64,<データ>` 。
</details>

### 問4

macOSで `base64 -i image.png` の後に `| tr -d '\n'` が必要な理由は？

<details>
<summary>答え</summary>

`base64` コマンドはデフォルトで76文字ごとに改行を入れて出力する。Data URI に改行が混ざると URL として壊れた扱いになり画像が表示されないため、`tr -d '\n'` で改行をすべて除去して1行にする。

- `|` — 左のコマンドの出力を右のコマンドに渡す（パイプ）
- `tr -d '\n'` — `tr` は文字を変換・削除するコマンド。`-d` は削除モード、`'\n'` は改行文字
</details>

### 問5

base64埋め込みが向いていないケースを1つ挙げよ。

<details>
<summary>答え</summary>
頻繁に変更する画像や大きい画像。base64化するとファイルサイズが約1.33倍になり、差分（diff）も読みにくくなるため。
</details>
