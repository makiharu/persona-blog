---
  title: "AWS IAM Identity CenterとAWS CLIのSSO認証を設定する"
  description: "AWS IAM Identity Centerを使ってAWS CLIからSSO認証する方法を整理する"
  categories: ["Tech"]
  date: 2026-09-20
  tags: [aws, iam-identity-center, aws-cli, sso, terraform, security]
  summary: "AWS IAM Identity Centerで作業用ユーザーを作成し、AWS CLIからSSO認証するまでの手順を整理します。"
---

## はじめに

Terraformを使ってAWSリソースを操作したかったので、その前準備としてTerraformのインストール、aws cliインストール、AWS IAM Identity Center（旧AWS SSO）で作業用ユーザーを作成し、AWS CLIからSSO認証できる状態にしました。その時のメモです。

## 1. AWS CLIをインストールする

macOSでは、Homebrewを使ってAWS CLIをインストールできます。

この記事では、実際に使用したHomebrew経由の方法でインストールします。AWS公式ドキュメントでは、macOS向けにインストールスクリプトやインストーラーを使う方法も案内されています。

```bash
brew install awscli
```

インストール後、バージョンを確認します。バージョン確認には`aws -v`ではなく`aws --version`を使います。

```bash
aws --version
```

出力例：

```text
aws-cli/2.33.17 Python/3.13.11 Darwin/25.5.0 exe/arm64
```

## 2. IAM Identity Centerでユーザーを作成する

AWS Management Consoleで、次の設定を行う。

1. IAM Identity Centerで作業用ユーザーを作成する(マルチアカウントのアクセス許可 > AWSアカウントから作成)
2. 対象のAWSアカウントにユーザーを割り当てる
3. 必要なPermission setを割り当てる

検証目的なので、今回は`AdministratorAccess`を使用したが、実際の運用では必要最小限の権限を持つPermission setを用意すること。※権限の付け替えはまた別の機会に行ってみる。

これにより、日常の作業でAWSルートユーザーを使わずに済む。

## 3. AWS CLIでSSO認証を設定する

```bash
aws configure sso
```

対話形式で、次の項目を入力する。

```text
SSO session name：任意のセッション名
SSO start URL：https://<organization>.awsapps.com/start　// ※
SSO region：ap-northeast-1
SSO registration scopes：空欄のままEnter
```

※sso start URLは、作成したユーザーでAWS account portalを開いた時のURL。
![alt text](image-1.png)

ブラウザが開いたら、IAM Identity Centerのユーザーでログインする。続いて、使用するAWSアカウントとPermission setを選択する。

最後に、分かりやすいプロファイル名を設定する。

```text
Profile name：<設定したプロファイル名>
```

ここで指定するプロファイル名は任意の名前である。以降のコマンドでは、`YOUR_PROFILE_NAME`を自分で設定した名前に置き換える。

## 4. 認証を確認する

```bash
aws sts get-caller-identity --profile YOUR_PROFILE_NAME
```


```json
{
  "UserId": "<user-id>",
  "Account": "<account-id>",
  "Arn": "arn:aws:sts::<account-id>:assumed-role/<role-name>/<user-name>"
}
```

## 5. SSOセッションが切れた場合

Permission Setのセッション時間は2時間に設定した。AWS CLIは、IAM Identity Centerへのログインセッションが有効な間はAWS認証情報を必要に応じて自動更新する。SSOログインセッション自体が切れた場合は、aws sso loginを再実行する。

```bash
aws sso login --profile YOUR_PROFILE_NAME
```

## 6. TerraformでAWSプロファイルを使う際には

```bash
export AWS_PROFILE=YOUR_PROFILE_NAME
aws sts get-caller-identity
```

この状態でTerraformを実行すると、指定したプロファイルの認証情報が使用される。

## セットアップチェックリスト

- [ ] AWS CLIをインストールし、`aws --version`でバージョンを確認した
- [ ] IAM Identity Centerで作業用ユーザーを作成した
- [ ] 対象のAWSアカウントにユーザーを割り当てた
- [ ] 必要なPermission setを割り当てた
- [ ] `aws configure sso`でSSO設定を完了した
- [ ] `aws sso login --profile YOUR_PROFILE_NAME`でログインした
- [ ] `aws sts get-caller-identity --profile YOUR_PROFILE_NAME`で対象アカウントを確認した
- [ ] 必要に応じて`AWS_PROFILE`を設定した

## 設定時のポイント

- 日常の作業では、AWSルートユーザーを使用しない
- Permission setは、用途に必要な最小限の権限で作成する
- 検証で`AdministratorAccess`を使った場合は、運用前に権限を見直す
- Terraformを実行する前に、`aws sts get-caller-identity`で対象アカウントを確認する
- SSOセッションが切れた場合は、`aws sso login`を再実行する
- アクセスキーやシークレットキーを発行・保存せず、SSOによる一時認証情報を利用する
- AWSアカウントID、SSO start URL、ARN、ユーザー情報などを公開リポジトリに載せない

## 理解度チェック

### 問1

IAM Identity Centerを使う主なメリットは何か？

<details>
<summary>答え</summary>
ユーザーとAWSアカウントへのアクセス権を一元管理し、長期的なアクセスキーを使わずに、一時的な認証情報でAWSへアクセスできること。
</details>

### 問2

Permission setは何を定義するものか？

<details>
<summary>答え</summary>
ユーザーやグループがAWSアカウント内で持つ権限のセット。ユーザーやグループに割り当てることで、対象アカウントに必要なアクセス権を付与する。
</details>

### 問3

AWS CLIでIAM Identity CenterのSSO設定を始めるコマンドは？

<details>
<summary>答え</summary>

```bash
aws configure sso
```
</details>

### 問4

`aws sts get-caller-identity`を実行すると何を確認できるか？

<details>
<summary>答え</summary>
現在のAWS認証情報で、どのユーザーまたはロールとして、どのAWSアカウントに接続しているかを確認できる。Terraformを実行する前のアカウント確認にも使える。
</details>

### 問5

SSOセッションが切れた場合は、どのコマンドを実行するか？

<details>
<summary>答え</summary>

```bash
aws sso login --profile YOUR_PROFILE_NAME
```
</details>

### 問6

検証環境で`AdministratorAccess`を使った場合、運用前に何を見直すべきか？

<details>
<summary>答え</summary>
実際の用途に必要な最小限の権限を持つPermission setへ見直す。管理者権限を常用しないことが重要。
</details>


## 参考リンク

- [AWS CLIのインストール（Homebrew Formulae）](https://formulae.brew.sh/formula/awscli)
- [AWS CLIのインストール・更新（AWS公式）](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [AWS CLIでIAM Identity Centerを設定する（AWS公式）](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)
- [AWS IAM Identity CenterでユーザーまたはグループにAWSアカウントへのアクセス権を割り当てる（AWS公式）](https://docs.aws.amazon.com/singlesignon/latest/userguide/assignusers.html)
- [AWSアカウントへのアクセスを設定する（AWS公式）](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-accounts.html)
- [Permission setを作成する（AWS公式）](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtocreatepermissionset.html)


## ひとこと

実際にIAM Identity Centerを使って、ユーザー作成を行い、Permission setの割り当てまでやってみたのは勉強になった。以前、[AWSでMFAを強制する方法とIAM Identity Centerの使い分け](https://makiharu.github.io/persona-blog/tech/02/)で整理した内容を、今回はハンズオンで試した形になった。
最近、名前をよく聞くTerraformを勉強してみようというところからのスタートなので、今回作成したユーザーを使って、AWSリソースをいじってみるようにしたい。


---
