# ローカルJenkins導入手順

Docker DesktopとWSL 2が未導入の場合は、先に[Docker Desktop・WSL導入手順](Docker%20Desktop・WSL導入手順.md)を実施してください。

Docker Desktop上にJenkinsを構築し、EC2上の各Webアプリをジョブから操作できるようにする手順です。Jenkins自体はローカルPCで動き、アプリのビルド・起動はEC2上で行います。

## 前提

- WindowsにDocker Desktopをインストール済み
- Docker DesktopはLinux containersで起動
- EC2へ手動SSH接続できる
- 共通秘密鍵 `ec2-key-1.pem` を保管している
- EC2のSecurity Groupで、開発PCからTCP 22番へ接続できる

## 1. Docker Desktopを確認

PowerShellで実行します。

```powershell
docker version
docker compose version
```

`docker` が見つからない場合はDocker Desktopを起動し、SettingsのWSL 2 based engineを有効にします。

## 2. Jenkinsを起動

Jenkinsの設定をDocker volumeへ保存します。コンテナを削除してもジョブ設定やCredentialsを維持できます。

```powershell
docker volume create jenkins_home
docker run -d --name jenkins --restart unless-stopped `
  -p 127.0.0.1:8080:8080 -p 50000:50000 `
  -v jenkins_home:/var/jenkins_home `
  jenkins/jenkins:lts-jdk21
docker ps --filter name=jenkins
docker logs jenkins
```

ブラウザで `http://localhost:8080` を開き、ログに表示された初期パスワードを入力します。

## 3. 初期設定

1. Install suggested pluginsを選択する。
2. Pipeline、Git、SSH Agentプラグインが入っていることを確認する。
3. 管理者ユーザーを作成する。
4. Jenkins URLを `http://localhost:8080/` にする。

Jenkinsを外部公開する必要はありません。`-p 127.0.0.1:8080:8080` により、このPCからだけアクセスできます。

## 4. 認証情報をCredentialsへ登録

### 4.1. SSH鍵（アプリ操作用）

1. Manage Jenkins → Credentials → System → Global credentialsへ進む。
2. Add CredentialsでKindに「SSH Username with private key」を選ぶ。
3. Usernameへ `ec2-user` を入力する。
4. IDへ `ec2-key-1` を入力する。
5. Private Keyへ `ec2-key-1.pem` の内容を登録する。

秘密鍵をPipelineへ直接記載したり、Gitへコミットしたりしません。登録後は画面から秘密鍵を再表示できないため、バックアップを安全に保管します。

### 4.2. AWS認証情報（EC2電源操作用）

EC2の起動・停止をJenkinsから制御する場合に登録します。

1. Manage Jenkins → Credentials → System → Global credentialsへ進む。
2. Add CredentialsでKindに「Username with password」（または「AWS Credentials」）を選ぶ。
3. Username（Access Key ID）へAWSアクセスキーIDを入力する。
4. Password（Secret Access Key）へAWSシークレットアクセスキーを入力する。
5. IDへ `aws-credentials` を入力して保存する。

### 4.3. JenkinsコンテナへのAWS CLI導入

EC2電源操作ジョブでAWS CLIを使用するため、Jenkinsコンテナ内にAWS CLIを導入します。

```powershell
docker exec -u 0 jenkins apt-get update
docker exec -u 0 jenkins apt-get install -y awscli
docker exec jenkins aws --version
```

## 5. Pipelineジョブを作成

### 5.1. アプリデプロイ・操作ジョブ

1. New Item → Pipelineを選択し、ジョブ名（例: `app-deploy`）を入力する。
2. 「This project is parameterized」にチェックを入れ、`Jenkins運用.md` のパラメータ（`PROJECT`、`ACTION`、`EC2_HOST`）を設定する。
3. Pipeline definitionをPipeline scriptにする。
4. `Jenkins運用.md` の「アプリ操作Pipelineの構成例」を登録する。
5. 保存後、Build with Parametersから `PROJECT=crane-rank`、`ACTION=start`、`EC2_HOST=<EC2のホスト名>` で実行する。

### 5.2. EC2起動・停止ジョブ

1. New Item → Pipelineを選択し、ジョブ名（例: `ec2-power`）を入力する。
2. 「This project is parameterized」にチェックを入れ、`Jenkins運用.md` のパラメータ（`ACTION`、`TARGET_NAME`、`AWS_REGION`）を設定する。
3. Pipeline definitionをPipeline scriptにする。
4. `Jenkins運用.md` の「EC2起動・停止Pipelineの構成例」を登録する。
5. 保存後、Build with Parametersから `ACTION=status`、`TARGET_NAME=ec2-1` で実行して状態確認ができることを確認する。

## 6. 動作確認

```powershell
docker logs --tail 100 jenkins
ssh -i .\ec2-key-1.pem ec2-user@<EC2のホスト名> "docker ps"
```

Jenkinsのコンソール出力にSSH接続成功とCompose操作が記録され、対象アプリのhealth checkが成功すれば導入完了です。

## 7. 日常操作・停止・バックアップ

```powershell
docker stop jenkins
docker start jenkins
docker logs --tail 100 jenkins
```

ジョブ設定とCredentialsは `jenkins_home` に保存されます。Docker volumeのバックアップ、Jenkinsの管理者パスワード管理、プラグイン更新を定期的に行います。Jenkinsを完全に削除する場合でも、先にvolumeをバックアップしてください。
