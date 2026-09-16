# Docker Desktop・WSL 2導入手順

Docker Desktop上でJenkinsを動かすためのWindows側の準備手順です。管理者権限が必要な操作があります。

## 前提

- Windows 10 version 22H2以降、またはWindows 11
- BIOS/UEFIで仮想化支援機能（Intel VT-x / AMD-V）が有効
- Docker Desktopを利用できる権限

## 1. WSLをインストール

管理者としてPowerShellを起動し、実行します。

```powershell
wsl --install
```

再起動を求められたらWindowsを再起動します。再起動後、Ubuntuの初回画面でLinuxユーザー名とパスワードを作成します。これはWindowsのパスワードとは別のものです。

既にWSLを導入済みの場合は、状態を確認します。

```powershell
wsl --status
wsl --version
wsl -l -v
```

## 2. WSL 2を既定に設定・更新

```powershell
wsl --update
wsl --set-default-version 2
wsl -l -v
```

ディストリビューションのVERSIONが1の場合は、ディストリビューション名を確認して2へ変換します。

```powershell
wsl --set-version Ubuntu 2
```

`Ubuntu` は `wsl -l -v` に表示された実際の名前へ置き換えます。

## 3. Docker Desktopをインストール

Docker公式サイトからDocker Desktop for Windowsをダウンロードしてインストールします。インストール時はWSL 2 based engineを選択します。

インストール後、Docker Desktopを起動し、Settingsで次を確認します。

1. General → Use the WSL 2 based engine：有効
2. Resources → WSL Integration → Enable integration with my default WSL distro：有効
3. 必要ならUbuntuのトグルも有効
4. Apply & Restartを実行

Docker Desktopの設定画面にWSL Integrationが表示されない場合は、Docker Desktopを最新版へ更新し、WSL 2が正常に動作することを確認します。

## 4. 動作確認

PowerShellで実行します。

```powershell
docker version
docker compose version
docker run --rm hello-world
```

次にUbuntu側でも確認します。

```powershell
wsl -d Ubuntu
```

Ubuntuのシェルで次を実行します。

```bash
docker version
docker compose version
docker run --rm hello-world
exit
```

`hello-world` が成功すれば、Docker CLIからDocker Desktopのエンジンへ接続できています。

## 5. Jenkins導入前の確認

Docker Desktopを起動した状態で、Jenkins導入手順へ進みます。

```powershell
docker volume create jenkins_home
docker ps
```

WSL上のプロジェクトをDockerから扱う場合は、Windowsファイルシステム配下よりWSLのLinuxファイルシステム配下の方が高速です。ただし今回のJenkins構成では、Jenkinsの設定をDocker volumeへ保存し、EC2操作はSSH経由で行います。

## トラブルシューティング

### `wsl --install` が失敗する

Windows Updateを適用して再起動します。企業PCなどで機能が制限されている場合は、Windowsの「仮想マシンプラットフォーム」と「Linux用Windowsサブシステム」を有効化してから再実行します。

### `docker` が見つからない

Docker Desktopを起動し、PowerShellを開き直します。`where.exe docker` でDocker CLIのパスも確認できます。

### Docker Desktopが起動しない

BIOS/UEFIの仮想化設定、WSLの状態（`wsl --status`）、Docker DesktopのDiagnosticsを確認します。WSLディストリビューションがVERSION 2であることも確認します。

### Jenkinsコンテナが起動しない

```powershell
docker logs jenkins
docker ps -a --filter name=jenkins
```

8080番ポートが使用中なら、`-p 127.0.0.1:8081:8080` のようにホスト側ポートを変更し、アクセスURLも合わせます。
