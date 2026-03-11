---
date: '2026-02-08T10:00:00+09:00'
draft: false
title: 'S3署名付きURL入門 - 期限付きで安全にファイルを共有する方法'
categories: ["Tech"]
tags: ["AWS", "S3", "署名付きURL", "クラウド"]
summary: "AWS S3の署名付きURL（Presigned URL）を使って、期限付きで安全にファイルを共有する方法を初心者向けに解説します。"
---

## はじめに

こんな状況はありませんか？

> NotebookLMで作った動画を友人に共有したい。でも、YouTubeに上げるほどでもないし、誰でも見れる状態にはしたくない...

この記事では、**AWS S3の署名付きURL（Presigned URL）** を使って、期限付きで安全にファイルを共有する方法を解説します。

### 今回のユースケース

- NotebookLMで作成した動画を友人数人に共有したい
- アクセス数は少ない（数人程度）
- 閲覧期間は短めでよい（数日〜1週間）
- 不特定多数には公開したくない

このような場合、署名付きURLがぴったりです。

## 署名付きURLとは？

**署名付きURL（Presigned URL）** は、S3のプライベートなファイルに**一時的にアクセスできる特別なURL**です。

```
通常のS3 URL（アクセス不可）:
https://my-bucket.s3.ap-northeast-1.amazonaws.com/video.mp4
→ AccessDenied

署名付きURL（期限内ならアクセス可能）:
https://my-bucket.s3.ap-northeast-1.amazonaws.com/video.mp4
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=...
  &X-Amz-Date=20260208T000000Z
  &X-Amz-Expires=3600
  &X-Amz-Signature=abc123...
→ 動画が再生できる！
```

URLにアクセス権限の情報（署名）が埋め込まれているため、このURLを知っている人だけがファイルにアクセスできます。

## なぜ署名付きURLを使うのか？

### 他の方法との比較

| 方法 | メリット | デメリット |
|------|---------|-----------|
| S3を公開設定にする | 簡単 | 誰でもアクセスできてしまう |
| CloudFront + 署名付きCookie | 高機能 | 設定が複雑 |
| **署名付きURL** | 簡単＆安全 | URLが長い、期限がある |

少人数への一時的な共有なら、署名付きURLが最もシンプルです。

### 署名付きURLのメリット

- **S3バケットは非公開のまま**でOK
- **期限を設定**できる（1秒〜7日間）
- **AWSアカウント不要**で相手に共有できる
- URLをLINEやメールで送るだけ

## 仕組み

署名付きURLの仕組みをざっくり説明します。

```
1. あなた（AWSアカウント所有者）
   ↓ 署名付きURLを生成

2. 署名付きURL
   - ファイルのパス
   - 有効期限
   - 署名（あなたの認証情報で作成）
   ↓ 友人にURLを送る

3. 友人がURLにアクセス
   ↓

4. S3が署名を検証
   - 署名は正しいか？
   - 期限内か？
   ↓ OKならファイルを返す

5. 友人が動画を視聴できる
```

重要なのは、**署名はあなたの認証情報で作成される**ということ。つまり、あなたがアクセス権を持つファイルだけURLを生成できます。

## 実際にやってみよう

### 前提条件

- AWSアカウントを持っている
- S3バケットにファイルをアップロード済み
- AWS CLIがインストール済み

### 方法1: AWS CLIで生成（最も簡単）

```bash
# 基本形
aws s3 presign s3://バケット名/ファイル名

# 有効期限を指定（秒単位、デフォルトは3600秒=1時間）
aws s3 presign s3://my-bucket/video.mp4 --expires-in 86400
```

`--expires-in 86400` で24時間（86400秒）有効なURLが生成されます。

```bash
# 実行例
$ aws s3 presign s3://my-videos/notebooklm-output.mp4 --expires-in 604800

https://my-videos.s3.ap-northeast-1.amazonaws.com/notebooklm-output.mp4?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIA...
```

生成されたURLを友人にLINEやメールで送れば完了です。

### 方法2: AWSコンソールから生成

1. AWSコンソールでS3を開く
2. 対象のファイルを選択
3. 「アクション」→「署名付きURLで共有」
4. 有効期限を設定して「署名付きURLを作成」

GUIで簡単に作成できます。

### 方法3: Python（boto3）で生成

プログラムから生成したい場合はSDKを使います。

```python
import boto3

s3_client = boto3.client('s3')

url = s3_client.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'my-bucket',
        'Key': 'video.mp4'
    },
    ExpiresIn=604800  # 7日間（秒）
)

print(url)
```

### 方法4: Node.js（AWS SDK v3）で生成

```javascript
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const client = new S3Client({ region: "ap-northeast-1" });

const command = new GetObjectCommand({
  Bucket: "my-bucket",
  Key: "video.mp4",
});

const url = await getSignedUrl(client, command, { expiresIn: 604800 });
console.log(url);
```

## 有効期限の目安

用途に応じて適切な有効期限を設定しましょう。

| 用途 | 推奨期限 | 秒数 |
|------|---------|------|
| その場で見せる | 1時間 | 3600 |
| 当日中に見てほしい | 24時間 | 86400 |
| 数日中に見てほしい | 3日間 | 259200 |
| 1週間以内に見てほしい | 7日間 | 604800 |

**注意**: 署名付きURLの最大有効期限は**7日間**です（IAMユーザーの認証情報を使う場合）。

## 注意点

### 1. URLを知っていれば誰でもアクセスできる

署名付きURLは「知っている人なら誰でも」アクセスできます。URLが漏れると意図しない人にも見られる可能性があります。

**対策**: 信頼できる相手にだけ共有し、有効期限を短めに設定する。

### 2. 一度発行したURLは取り消せない

URLを発行した後、「やっぱり取り消したい」と思っても、有効期限が切れるまで無効化できません。

**対策**: 有効期限を必要最小限にする。どうしても無効化したい場合はファイル自体を削除する。

### 3. ダウンロード回数は制限できない

署名付きURLでは「3回までダウンロード可能」のような制限はできません。

**対策**: 期限を短くするか、CloudFrontの署名付きURLを検討する。

### 4. 大量アクセスには向かない

署名付きURLは一時的な共有向けです。多数のユーザーに配信する場合は、CloudFrontとの組み合わせを検討しましょう。

## 料金について

署名付きURLの生成自体は**無料**です。

ただし、以下の料金は発生します。

- **S3ストレージ料金**: ファイルの保存容量に応じて
- **データ転送料金**: ダウンロードされた分だけ

少人数への共有なら、ほぼ無視できる金額です（数円〜数十円程度）。

## まとめ

署名付きURLのポイントをまとめます。

- **期限付き**でプライベートファイルを共有できる
- S3バケットは**非公開のまま**でOK
- AWS CLIなら `aws s3 presign` で簡単に生成
- 最大有効期限は**7日間**
- URLが漏れるとアクセスされるので、**信頼できる相手にだけ**共有

友人への動画共有など、少人数への一時的な共有にはぴったりの方法です。

## 理解度チェック

### 問1

署名付きURLを使うと、S3バケットの公開設定はどうなりますか？

<details>
<summary>答え</summary>
非公開のままでOK。署名付きURL自体にアクセス権限の情報が含まれているため、バケットを公開する必要はない。
</details>

### 問2

署名付きURLの最大有効期限は何日間ですか？

<details>
<summary>答え</summary>
7日間（604800秒）。IAMユーザーの認証情報を使う場合の制限。
</details>

### 問3

発行した署名付きURLを途中で無効化できますか？

<details>
<summary>答え</summary>
できない。有効期限が切れるまで有効。無効化したい場合はファイル自体を削除する必要がある。
</details>

### 問4

AWS CLIで署名付きURLを生成するコマンドは何ですか？

<details>
<summary>答え</summary>
`aws s3 presign s3://バケット名/ファイル名`。有効期限を指定する場合は `--expires-in 秒数` オプションを追加する。
</details>

### 問5

署名付きURLの生成自体に料金はかかりますか？

<details>
<summary>答え</summary>
かからない（無料）。ただし、S3ストレージ料金とダウンロード時のデータ転送料金は発生する。
</details>

### 問6

署名付きURLで「3回までダウンロード可能」のような回数制限はできますか？

<details>
<summary>答え</summary>
できない。署名付きURLでは期限内であれば何度でもアクセス可能。回数制限が必要な場合はCloudFrontの署名付きURLなど別の方法を検討する。
</details>

### 問7

署名付きURLが漏れた場合のリスクを軽減するための対策を2つ挙げてください。

<details>
<summary>答え</summary>
1. 有効期限を必要最小限に設定する。2. 信頼できる相手にだけURLを共有する。（他にも「どうしても無効化したい場合はファイル自体を削除する」なども可）
</details>
