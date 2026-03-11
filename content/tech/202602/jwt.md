---
date: '2026-02-04T23:13:45+09:00'
draft: false
title: 'JWT入門 - 初心者のためのJSON Web Token'
categories: ["Tech"]
tags: ["JWT", "認証", "セキュリティ", "Web開発"]
summary: "JWT（JSON Web Token）の仕組みを初心者向けにわかりやすく解説します。構造、使い方、セキュリティの注意点まで基礎から学べます。"
---

## はじめに

Webアプリケーションを開発していると、「JWT」という言葉をよく目にします。ログイン機能やAPI認証で使われる重要な技術ですが、最初は少しとっつきにくいかもしれません。

この記事では、JWTの基本を初心者向けにわかりやすく解説します。

## JWTとは？

**JWT（JSON Web Token）** は、情報を安全にやり取りするための**トークン形式**です。

簡単に言うと、「**誰が何の権限を持っているか**」を証明するデジタルな身分証明書のようなものです。

### なぜJWTが必要なの？

従来のWebアプリケーションでは、ログイン状態をサーバー側の「セッション」で管理していました。しかし、この方法にはいくつかの課題があります。

| 方式 | 特徴 |
|------|------|
| セッション方式 | サーバーがログイン状態を記憶する必要がある |
| JWT方式 | トークン自体に情報が含まれ、サーバーは記憶不要 |

JWTを使うと、サーバーは状態を持たなくて済むため（**ステートレス**）、複数のサーバーに分散させやすくなります。

## JWTの構造

JWTは3つの部分で構成されています。それぞれ **ドット（.）** で区切られています。

```
xxxxx.yyyyy.zzzzz
  ↓      ↓      ↓
Header.Payload.Signature
```

実際のJWTはこのような文字列です。

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.KMUFsIDTnFmyG3nMiGM6H9FNFUROf3wh7SmqJp-QV30
```

一見すると暗号のように見えますが、実は**Base64URL**でエンコードされているだけです。

### 1. Header（ヘッダー）

トークンのタイプと、署名に使うアルゴリズムを指定します。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- `alg`: 署名アルゴリズム（HS256, RS256など）
- `typ`: トークンのタイプ（JWT）

### 2. Payload（ペイロード）

実際に伝えたい情報（**クレーム**と呼ばれます）が含まれます。

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622
}
```

よく使われるクレーム（予約済み）:

| クレーム | 説明 |
|---------|------|
| `sub` | Subject（主題）- ユーザーIDなど |
| `iat` | Issued At（発行日時） |
| `exp` | Expiration（有効期限） |
| `iss` | Issuer（発行者） |
| `aud` | Audience（対象者） |

**注意**: Payloadは暗号化されていません。Base64でデコードすれば誰でも中身を見れます。パスワードなどの機密情報は絶対に入れないでください。

### 3. Signature（署名）

ヘッダーとペイロードを結合し、秘密鍵で署名したものです。

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

この署名により、トークンが**改ざんされていないこと**を検証できます。

## JWTの流れ

実際の認証フローを見てみましょう。

```
1. ユーザー: IDとパスワードでログイン
        ↓
2. サーバー: 認証OK → JWTを発行して返す
        ↓
3. クライアント: JWTを保存（localStorageやCookieなど）
        ↓
4. クライアント: APIリクエスト時にJWTを送信
   Authorization: Bearer eyJhbGciOi...
        ↓
5. サーバー: JWTの署名を検証 → OKならリクエストを処理
```

## JWTを実際に見てみよう

[jwt.io](https://jwt.io/) というサイトで、JWTをデコードして中身を確認できます。

試しに以下のトークンをデコードしてみてください。

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

## JWTのメリット・デメリット

### メリット

- **ステートレス**: サーバーがセッション情報を保持する必要がない
- **スケーラブル**: 複数サーバー間で共有しやすい
- **クロスドメイン**: 異なるドメイン間でも使いやすい
- **モバイルフレンドリー**: ネイティブアプリでも扱いやすい

### デメリット

- **トークンサイズ**: セッションIDより大きくなりがち
- **無効化が難しい**: 発行したトークンを途中で無効にするのが難しい
- **Payload丸見え**: 暗号化されていないので機密情報は入れられない

## セキュリティの注意点

JWTを安全に使うために、以下の点に注意しましょう。

### 1. 秘密鍵は厳重に管理する

秘密鍵が漏れると、誰でもトークンを偽造できてしまいます。

### 2. 有効期限を短くする

`exp`クレームで有効期限を設定し、できるだけ短くしましょう。長期間有効なトークンはリスクが高くなります。

### 3. HTTPSを使う

JWTは平文で送信されるため、必ずHTTPSで通信しましょう。

### 4. 機密情報をPayloadに入れない

Payloadは暗号化されていません。パスワードやクレジットカード番号などは絶対に入れないでください。

### 5. アルゴリズムを固定する

サーバー側で受け入れるアルゴリズムを明示的に指定しましょう。`alg: none`攻撃を防ぐためです。

## よくある質問

### Q: JWTはどこに保存すべき？

**A**: 用途によりますが、一般的には以下の選択肢があります。

| 保存場所 | 特徴 |
|---------|------|
| HttpOnly Cookie | XSS攻撃に強い。CSRF対策が必要 |
| localStorage | 使いやすいがXSS攻撃に弱い |
| メモリ（変数） | 最も安全だがページリロードで消える |

セキュリティを重視するなら**HttpOnly Cookie**がおすすめです。

### Q: JWTとセッションどちらを使うべき？

**A**: どちらが良いかは要件次第です。

- **JWT向き**: マイクロサービス、SPA、モバイルアプリ、サーバーレス
- **セッション向き**: 従来のWebアプリ、即座にログアウトさせたい場合

## まとめ

JWTの基本をまとめると以下の通りです。

- JWTは**Header.Payload.Signature**の3部構成
- Payloadに情報を含み、Signatureで改ざんを検知
- **ステートレス**な認証を実現できる
- Payloadは**暗号化されていない**ので機密情報は入れない
- **有効期限**と**HTTPS**でセキュリティを確保

JWTは現代のWeb開発において欠かせない技術です。基本を理解した上で、適切に活用していきましょう。

## 参考リンク

- [JWT.IO](https://jwt.io/) - JWTのデコード・検証ツール
- [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) - JWT の公式仕様


## 理解度チェック

理解度チェッククイズ（全5問）  
問1

JWTが「ステートレス」と呼ばれるのはなぜですか？

<details> <summary>答え</summary> サーバーがログイン状態を保持せず、トークン自体に必要な情報が含まれているため。 </details>  

問2

JWTのPayloadにパスワードなどの機密情報を入れてはいけない理由は何ですか？

<details> <summary>答え</summary> Payloadは暗号化されておらず、Base64デコードすれば誰でも中身を見られるため。 </details>  

問3

JWTのSignature（署名）は何を保証するためのものですか？

<details> <summary>答え</summary> トークンが発行後に改ざんされていないことを保証するため。 </details>  

問4

JWTを使った認証で、クライアントはAPIリクエスト時にトークンをどこに送りますか？

<details> <summary>答え</summary> HTTPヘッダーの `Authorization: Bearer <JWT>` に含めて送信する。 </details>  

問5

JWTに exp（有効期限）を設定するのはなぜですか？

<details> <summary>答え</summary> トークンが漏洩した場合の被害を最小限に抑えるため。 </details>  

問6

JWTは途中で無効化しづらいという欠点があります。
この問題を軽減するために、よく使われる設計は何でしょう？

<details> <summary>答え</summary> 有効期限を短くし、リフレッシュトークンと組み合わせて運用する。 </details>