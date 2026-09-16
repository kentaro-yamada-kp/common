# AWS共通運用

| 項目 | 方針 |
| --- | --- |
| AWSアカウント | 現在のアカウントを共通利用 |
| ローカルAWS CLI | `aws-dev1` |
| リージョン | `ap-northeast-1` |
| 実行基盤 | 原則1台のAmazon Linux 2023 EC2 |
| アプリ | プロジェクトごとのDocker Compose |
| EC2上のAWS認証 | IAMロール |

## 共通リソース名

既存リソースを改名するときは、削除・再作成ではなく、まずNameタグや表示名を変更します。実体の再作成が必要な場合は、バックアップと切替手順を先に用意します。

| リソース | 共通名 |
| --- | --- |
| EC2 Nameタグ | `ec2-1` |
| EBSボリューム | `vol-1` |
| EC2キーペア／ローカル秘密鍵 | `ec2-key-1` / `ec2-key-1.pem` |
| Security Group | 名称なしで運用 |
| EC2 IAMロール | 現在は未作成 |
| Elastic IP | `GIP-1` |
| Jenkins Credentials ID | `ec2-key-1` |

EC2のInstance ID（`i-...`）は改名できない識別子です。Jenkinsや手順書ではInstance IDを固定値として埋め込まず、NameタグまたはSSM・AWS CLIで取得します。

## 既存AWSリソースの改名・切替

変更前に、現在のInstance ID、Security Group、IAMロール、キーペア、Elastic IP、EBSを記録します。

- EC2の表示名はNameタグを `ec2-1` に変更できます。Instance IDは変わりません。
- Elastic IPなどの表示名もNameタグで変更できます。
- EC2キーペア名は変更できません。既存EC2へ接続できる状態で新しいキーペアを作成し、新しい公開鍵を `/home/ec2-user/.ssh/authorized_keys` へ追加して接続確認後、旧公開鍵を削除します。秘密鍵はAWSから再ダウンロードできません。
- Security Group名とIAMロール名は既存リソースを直接改名せず、新しい共通名で作成してルール・ポリシーを移行し、依存先を切り替えてから旧リソースを削除します。

現状は、EC2 Nameタグを `ec2-1`、EBSボリュームのNameタグを `vol-1`、Elastic IPのNameタグを `GIP-1` とします。Security Groupには名前を付けず、IAMロールも作成しません。BedrockやSSMなどEC2からAWS APIを利用する段階で、必要最小限のIAMロールを別途検討します。

## EC2上の配置

```text
/home/ec2-user/projects/
  crane-rank/
  app2/
```

既存の `crane-rank` が `/home/ec2-user/crane-rank` にある場合は、まず現行ジョブを維持します。新規プロジェクトから `/home/ec2-user/projects/<project>` を使い、`crane-rank` の移行はバックアップと停止時間を確保して別作業で行います。

プロジェクトごとにホスト公開ポート、Composeプロジェクト名、コンテナ名、ボリューム、`.env.production`、Nginx設定名、ドメインを分離します。外部公開はNginxを入口にし、各アプリをローカルバインドポートへ転送します。開発中や停止中のアプリはComposeを停止します。

```powershell
$env:AWS_PROFILE = 'aws-dev1'
$env:AWS_REGION = 'ap-northeast-1'
aws sts get-caller-identity
aws ec2 describe-instances --query "Reservations[].Instances[].{Id:InstanceId,State:State.Name,IP:PublicIpAddress}" --output table
```

## セキュリティ・料金

- EC2には必要最小限のIAMロールだけを付与する（BedrockやSSMは利用時だけ追加）。
- AWSキー、秘密鍵、`.env.production` はGitへコミットしない。
- Jenkinsの秘密鍵はCredentialsへ登録し、プロジェクトへコピーしない。
- EC2停止後もEBS、Elastic IP、CloudWatch、S3、Bedrock等の料金は残り得る。
- 共有EC2ではメモリ・ディスク・CPU・ポート競合を確認し、必要ならインスタンス分離を検討する。
