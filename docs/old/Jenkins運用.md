# Docker Desktop上のJenkins運用

Windows開発PCのDocker DesktopでJenkinsを起動し、ジョブのパラメータで対象プロジェクトと操作を選びます。JenkinsはSSH経由でEC2上の対象Composeだけを操作します。

```text
Jenkins (Docker Desktop) --SSH--> EC2:/home/ec2-user/projects/<project>
```

JenkinsコンテナにDockerソケットを接続する必要はありません。Jenkinsは外部公開せず、ホストのlocalhostだけでアクセスします。

## 初回準備

1. Docker DesktopをLinux containersで起動する。
2. Jenkinsの永続ボリュームを作成する。
3. Jenkins CredentialsにEC2接続用秘密鍵を「SSH Username with private key」として登録する（ID：`ec2-key-1`、ユーザー `ec2-user`）。
4. EC2のセキュリティグループのSSH送信元を必要最小限にする。

```powershell
docker volume create jenkins_home
docker run -d --name jenkins --restart unless-stopped -p 127.0.0.1:8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts-jdk21
docker logs jenkins
```

初回パスワードを画面へ入力し、Pipeline、Git、SSH Agentプラグインを導入します。初期パスワードや認証情報をログへ出力しません。

## ジョブ一覧

Jenkins上では、役割と影響範囲に応じて以下の2つのジョブを用意します。

1. **アプリ操作ジョブ (`app-deploy`)**: アプリのデプロイ、開始、停止（SSH経由）
2. **EC2電源管理ジョブ (`ec2-power`)**: EC2インスタンス自体の起動、停止、状態確認（AWS CLI経由）

---

## 1. アプリ操作ジョブ (`app-deploy`)

### パラメータ

| パラメータ | 例 | 内容 |
| --- | --- | --- |
| `PROJECT` | `crane-rank` | 対象プロジェクト |
| `ACTION` | `deploy` / `start` / `stop` | 実行操作 |
| `BRANCH` | `main` | デプロイ対象ブランチ |
| `EC2_HOST` | 固定ホスト名 / IP | EC2接続先 |

プロジェクトと操作はchoiceで許可値を固定し、任意のシェル文字列を受け取りません。同じEC2を共有するため、ジョブの同時実行を禁止します。

### Pipelineの構成例

```groovy
pipeline {
  agent any
  options { disableConcurrentBuilds() }
  parameters {
    choice(name: 'PROJECT', choices: ['crane-rank', 'app2'], description: '操作対象プロジェクト')
    choice(name: 'ACTION', choices: ['deploy', 'start', 'stop'], description: '実行操作')
    string(name: 'EC2_HOST', defaultValue: 'ec2.example.com', description: 'EC2接続先ホスト名またはIP')
  }
  stages {
    stage('Operate App') {
      steps {
        sshagent(credentials: ['ec2-key-1']) {
          sh '''
            set -eu
            case "$PROJECT" in
              crane-rank) REMOTE_DIR=/home/ec2-user/crane-rank ;;
              app2) REMOTE_DIR=/home/ec2-user/projects/app2 ;;
              *) echo "invalid project" >&2; exit 1 ;;
            esac
            case "$ACTION" in
              deploy) CMD='git pull --ff-only && docker compose -f docker-compose.production.yml up -d --build --remove-orphans' ;;
              start) CMD='docker compose -f docker-compose.production.yml up -d' ;;
              stop) CMD='docker compose -f docker-compose.production.yml stop' ;;
              *) echo "invalid action" >&2; exit 1 ;;
            esac
            ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new ec2-user@"$EC2_HOST" "cd '$REMOTE_DIR' && $CMD"
          '''
        }
      }
    }
  }
}
```

これは構成例です。実運用ではプロジェクトごとにComposeファイル、リモートディレクトリ、health checkを固定設定します。`deploy` 後のhealth checkを必須にし、失敗時はComposeログを確認します。

---

## 2. EC2電源管理ジョブ (`ec2-power`)

EC2の起動・停止・状態確認を行う独立したジョブです。不要な時間帯にEC2を停止してクラウドコストを最小化しつつ、利用時にはワンクリックで起動して作業環境を迅速に復旧できます。

### パラメータ

| パラメータ | 規定値 / 選択肢 | 内容 |
| --- | --- | --- |
| `ACTION` | `status` / `start` / `stop` | 実行操作（status: 状態取得, start: 起動, stop: 停止） |
| `TARGET_NAME` | `ec2-1` | 対象EC2のNameタグ（Instance IDを直接埋め込まずタグで解決） |
| `AWS_REGION` | `ap-northeast-1` | AWSリージョン |

### Pipelineの構成例

```groovy
pipeline {
  agent any
  options { disableConcurrentBuilds() }
  parameters {
    choice(name: 'ACTION', choices: ['status', 'start', 'stop'], description: '実行する電源操作')
    string(name: 'TARGET_NAME', defaultValue: 'ec2-1', description: '操作対象のEC2 Nameタグ')
    string(name: 'AWS_REGION', defaultValue: 'ap-northeast-1', description: 'AWSリージョン')
  }
  stages {
    stage('EC2 Power Control') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'aws-credentials',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''
            set -eu
            export AWS_DEFAULT_REGION="$AWS_REGION"

            echo "=== 対象インスタンス検索 (Name: $TARGET_NAME) ==="
            INSTANCE_ID=$(aws ec2 describe-instances \
              --region "$AWS_REGION" \
              --filters "Name=tag:Name,Values=$TARGET_NAME" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
              --query "Reservations[0].Instances[0].InstanceId" \
              --output text)

            if [ "$INSTANCE_ID" = "None" ] || [ -z "$INSTANCE_ID" ]; then
              echo "エラー: Nameタグ '$TARGET_NAME' を持つEC2インスタンスが見つかりません。" >&2
              exit 1
            fi

            STATE=$(aws ec2 describe-instances \
              --instance-ids "$INSTANCE_ID" \
              --region "$AWS_REGION" \
              --query "Reservations[0].Instances[0].State.Name" \
              --output text)

            echo "Instance ID : $INSTANCE_ID"
            echo "Current State: $STATE"

            case "$ACTION" in
              status)
                PUBLIC_IP=$(aws ec2 describe-instances \
                  --instance-ids "$INSTANCE_ID" \
                  --region "$AWS_REGION" \
                  --query "Reservations[0].Instances[0].PublicIpAddress" \
                  --output text)
                echo "----------------------------------------"
                echo "ステータス確認結果:"
                echo "  Name      : $TARGET_NAME"
                echo "  ID        : $INSTANCE_ID"
                echo "  State     : $STATE"
                echo "  Public IP : $PUBLIC_IP"
                echo "----------------------------------------"
                ;;
              start)
                if [ "$STATE" = "running" ]; then
                  echo "EC2は既に起動しています (State: running)。追加操作は不要です。"
                elif [ "$STATE" = "pending" ]; then
                  echo "EC2は現在起動処理中です (State: pending)。完了までお待ちください。"
                elif [ "$STATE" = "stopping" ]; then
                  echo "エラー: EC2は停止処理中です。完全に stopped 状態になってから起動してください。" >&2
                  exit 1
                elif [ "$STATE" = "stopped" ]; then
                  echo "EC2 ($INSTANCE_ID) の起動リクエストを発行します..."
                  aws ec2 start-instances --instance-ids "$INSTANCE_ID" --region "$AWS_REGION"
                  echo "起動リクエストを送信しました。起動完了後、必要に応じてアプリ操作ジョブでコンテナ状態をご確認ください。"
                else
                  echo "エラー: 現在の状態 ($STATE) では起動を実行できません。" >&2
                  exit 1
                fi
                ;;
              stop)
                if [ "$STATE" = "stopped" ]; then
                  echo "EC2は既に停止済みです (State: stopped)。"
                elif [ "$STATE" = "stopping" ]; then
                  echo "EC2は現在停止処理中です (State: stopping)。"
                elif [ "$STATE" = "pending" ]; then
                  echo "エラー: EC2は起動処理中です。状態が安定してから停止してください。" >&2
                  exit 1
                elif [ "$STATE" = "running" ]; then
                  echo "EC2 ($INSTANCE_ID) の停止リクエストを発行します..."
                  aws ec2 stop-instances --instance-ids "$INSTANCE_ID" --region "$AWS_REGION"
                  echo "停止リクエストを送信しました。EC2本体の稼働課金は停止しますが、EBSボリュームやElastic IP等の保持コストは継続します。"
                else
                  echo "エラー: 現在の状態 ($STATE) では停止を実行できません。" >&2
                  exit 1
                fi
                ;;
              *)
                echo "エラー: 不正なアクション ($ACTION) です。" >&2
                exit 1
                ;;
            esac
          '''
        }
      }
    }
  }
}
```

### 安全設計・利点

- **二重操作防止**: インスタンスの現在状態（`running`, `stopped`, `pending`, `stopping`）を事前チェックし、誤った連続実行や中途半端な状態での誤操作を自動ガードします。
- **インスタンスID非固定化**: Nameタグ（`ec2-1`）をキーに動的にInstance IDを取得するため、インスタンス再作成時にもジョブ定義の書き換えが不要です。
- **リソース効率**: 開発停止時や夜間にEC2を停止することで無駄な稼働コストを抑え、必要なときのみ起動してチーム全体の費用対効果を最大化できます。

## 運用ルール

- `stop` はアプリだけを停止し、EC2停止は別の承認済みジョブに分ける。
- `deploy` はブランチ、対象、実行者、結果をJenkinsに記録する。
- DB削除・AIリセットなど破壊的操作は通常デプロイから分離し、手動承認を付ける。
- Jenkins自身のバックアップ、アップデート、管理者認証、ログ保持期間を決める。
- LinuxコンテナのJenkinsからWindows用 `.ps1` を呼ぶ場合は、PowerShell 7を入れたWindowsエージェントを別途用意する。
