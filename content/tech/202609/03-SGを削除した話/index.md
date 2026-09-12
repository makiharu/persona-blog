---
  title: "AWSセキュリティグループの削除前に確認すること"
  description: "技術関連の記事"
  date: 2026-09-13
  tags: [aws, security-group, security]
---

AWSのセキュリティグループ（Security Group、以下SG）を整理するときに、どのような確認が必要なのかを学習用・検証用の環境を想定して整理する。


## はじめに

SGは、EC2などのAWSリソースに対する通信を制御する仮想ファイアウォールです。SGにはインバウンドルールとアウトバウンドルールがあり、どの送信元からどのポートへの通信を許可するかを定義します。

SGを整理していると、広い範囲からのアクセスを許可するルールや、用途が分からないSGを見つけることがあります。しかし、ルールが広いからといって、すぐに削除してよいとは限りません。

削除前には、次の2つを分けて考える必要があります。

- その通信ルールは業務上必要なのか
- そのSGは現在どのリソースに関連付けられているのか

## `0.0.0.0/0`とは

`0.0.0.0/0`は、すべてのIPv4アドレスを表すCIDRです。SGのインバウンドルールでこの値を指定すると、指定したポートへの通信を、送信元IPアドレスによらず許可することになります。

たとえば、次のルールは世界中のIPv4アドレスからSSH接続を許可します。

| プロトコル | ポート | 送信元 | 意味 |
| --- | ---: | --- | --- |
| TCP | 22 | `0.0.0.0/0` | すべてのIPv4アドレスからSSHを許可 |

このような設定では、インターネット上のスキャンや攻撃の対象になる可能性が高くなります。特にSSH（22番）やRDP（3389番）、データベースのポートなどを全世界に公開するのは危険です。

一方で、公開WebサイトのHTTP（80番）やHTTPS（443番）では、利用者の送信元IPを事前に限定できないため、`0.0.0.0/0`を使うことがあります。この場合も、Webサーバーやロードバランサーなど、公開が意図されたリソースにだけ関連付けることが重要です。

また、今回問題にしたいのは主にインバウンドルールです。アウトバウンドの`0.0.0.0/0`は、リソースからインターネット上のサービスへ接続するために使われることがあり、インバウンドと同じ意味で単純に判断できません。

## 未使用のSGを整理する理由

未使用のSGを残しておくと、今すぐ通信が発生するわけではありません。しかし、将来そのSGを誤ってEC2やENIなどに関連付けたとき、広い範囲からのアクセスを許可する設定が意図せず有効になる可能性があります。

また、不要なSGが多いと、名前やルールを見ただけでは「現在使われている設定なのか」「過去の作業の残りなのか」を判断しにくくなります。これは、設定ミスや変更時の事故につながります。

そのため、未使用であることを確認できたSGは、削除または廃止候補として整理する価値があります。

## 本当に使われていないかを確認する

SGを削除する前に、少なくとも次の観点で確認します。

1. EC2インスタンスやENIに関連付けられていないか
2. 他のSGのルールから参照されていないか
3. ロードバランサー、RDS、ECSなどのAWSサービスで使われていないか
4. Auto Scaling、起動テンプレート、CloudFormationなどで指定されていないか
5. TerraformやAWS CDKなどのIaCで管理されていないか
6. 今後必要になる予定のSGではないか

### SGのルールを確認する

まず、対象SGの所属VPCやルールを確認します。以下は実際の環境で実行する場合の形式例です。値は自分の確認対象に置き換えます。

```bash
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxxxxxxxxxxx \
  --profile <profile> \
  --region <region>
```

`IpPermissions`がインバウンドルール、`IpPermissionsEgress`がアウトバウンドルールです。`0.0.0.0/0`だけでなく、`::/0`というIPv6の全アドレスを表すルールも確認します。

### ENIへの関連付けを確認する

SGはEC2インスタンスだけでなく、Elastic Network Interface（ENI）に関連付けられます。ロードバランサーやRDS、ECSなど、AWSサービスが管理するENIに関連付けられていることもあります。

```bash
aws ec2 describe-network-interfaces \
  --filters Name=group-id,Values=sg-xxxxxxxxxxxxxxxxx \
  --query 'NetworkInterfaces[].{ENI:NetworkInterfaceId,Status:Status,Type:InterfaceType,Description:Description,Instance:Attachment.InstanceId}' \
  --profile <profile> \
  --region <region>
```

結果が空であれば、少なくとも指定したSGを持つENIは見つかりませんでした。ただし、ENIがないことだけで、IaCやAWSサービスの設定から完全に不要だと断定することはできません。

### 他のSGから参照されていないか確認する

SGは、IPアドレスではなく別のSGを送信元として指定できます。たとえば、アプリケーションサーバーのSGからデータベースのSGへの通信を許可する構成です。

次のコマンドで、対象SGを参照しているSGを確認します。

```bash
aws ec2 describe-security-groups \
  --filters Name=ip-permission.group-id,Values=sg-xxxxxxxxxxxxxxxxx \
  --query 'SecurityGroups[].{GroupId:GroupId,GroupName:GroupName,VpcId:VpcId}' \
  --profile <profile> \
  --region <region>
```

ここで結果が返る場合、対象SGを削除すると参照元のルールも成立しなくなるため、先に参照関係を確認します。

### IaCや設定ファイルから参照されていないか確認する

リポジトリを管理している場合は、SG IDやSG名を検索します。

```bash
rg 'sg-xxxxxxxxxxxxxxxxx|対象SGの名前' .
```

ただし、SG IDを直接書かずにTerraformのリソース参照やCloudFormationのパラメータで渡している場合もあります。そのため、文字列検索だけでなく、IaCの依存関係やデプロイ設定も確認する必要があります。

## 安全に削除する

関連付けや参照がなく、IaCや運用上も不要だと判断できた場合に削除します。実環境で実行する前に、対象のアカウント・リージョン・SG IDが正しいことを確認します。

```bash
aws ec2 delete-security-group \
  --group-id sg-xxxxxxxxxxxxxxxxx \
  --profile <profile> \
  --region <region>
```

削除後は、対象SGが取得できなくなったことを確認します。

```bash
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxxxxxxxxxxx \
  --profile <profile> \
  --region <region>
```

削除済みであれば、通常は`InvalidGroup.NotFound`のようなエラーになります。

本番環境での削除に不安がある場合は、まずSGにタグを付けて廃止候補として記録し、一定期間利用されていないことを確認してから削除する方法もあります。

## おまけ：AWS CLIのプロファイル一覧を確認する

AWS CLIでは、複数のAWSアカウントや環境をプロファイルとして切り替えて利用できます。登録済みのプロファイル一覧は、次のコマンドで確認できます。

```bash
aws configure list-profiles
```

たとえば、次のようなプロファイルが表示されます。

```text
default
dev
production
```

コマンドを実行するときは、`--profile`で使用するプロファイルを指定します。

```bash
aws sts get-caller-identity --profile dev
```

削除や変更を行う前に、意図したAWSアカウントへ接続しているかを`aws sts get-caller-identity`で確認すると、対象アカウントを間違える事故を防ぎやすくなります。

## `DependencyViolation`が出た場合

SGがリソースや別のSGから参照されている状態で削除しようとすると、`DependencyViolation`が発生することがあります。

このエラーが出た場合は、削除コマンドを繰り返すのではなく、次の依存関係を再確認します。

- EC2インスタンスやENIへの関連付け
- ロードバランサーやRDS、ECSなどのサービス管理リソース
- 別のSGのインバウンドルールからの参照
- 起動テンプレートやAuto Scalingの設定
- CloudFormation、Terraform、CDKなどのIaC

AWS側で削除を拒否してくれることは安全装置になりますが、「削除できないから使われている場所が明らか」とは限りません。依存関係を特定してから、必要なものを付け替えるか、不要な参照を削除します。

## 今回の整理

SGを整理するときは、`0.0.0.0/0`があるかどうかだけで、機械的に削除対象を決めないことが重要です。

次の順番で確認すると、判断しやすくなります。

1. インバウンド・アウトバウンドのルールを確認する
2. 公開が意図されたポートか確認する
3. ENIやAWSサービスへの関連付けを確認する
4. 他のSGからの参照を確認する
5. IaCや起動設定からの参照を確認する
6. 廃止候補として記録し、必要なら一定期間観察する
7. 不要と判断できたSGを削除する

## まとめ

- `0.0.0.0/0`はすべてのIPv4アドレスを表す
- `0.0.0.0/0`のインバウンドルールは、指定ポートをインターネット全体に公開する設定になり得る
- ただし、HTTPやHTTPSなど、意図的な公開が必要なケースもある
- SGの削除前には、ENI、他のSG、AWSサービス、IaCを確認する
- SG IDの文字列検索だけでは不十分で、AWS上の関連付けも確認する必要がある
- 未使用のSGは、廃止候補として記録してから削除すると安全性を高められる

## 参考

- [AWS VPC セキュリティグループの操作](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-security-groups.html)
- [AWS CLI `describe-network-interfaces`](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-network-interfaces.html)
- [AWS Security Hub EC2セキュリティコントロール](https://docs.aws.amazon.com/securityhub/latest/userguide/ec2-controls.html)
