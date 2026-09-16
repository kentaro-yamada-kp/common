# 10.projects 共通基盤

## 概要

複数プロジェクトで共用するWSL、Docker、Jenkins、AWSの構築・運用情報とPipelineを管理します。

## 目次

- [環境構築](docs/10.setup/10.環境構築リスト.md)
- [運用](docs/20.operations/00.運用リスト.md)
- [ToDo](docs/99.ToDo.md)

## 管理範囲

このリポジトリは個別アプリに依存しない開発・運用基盤を管理します。AWSアカウント、EC2の電源操作、Docker Desktop、Jenkins本体、共通Nginx設定、共通Pipelineが対象です。アプリの環境変数、Compose構成、DB操作、固有機能は各プロジェクトで管理します。

    10.projects/
    ├─ common/             共通基盤、共通Nginx、共通Pipeline、共通ドキュメント
    ├─ crane-rank/         アプリ、固有Pipeline、固有ドキュメント
    └─ generic-matching/   アプリ、固有機能、固有ドキュメント

## 基本方針

- 秘密鍵、AWSキー、環境変数ファイルはGitへ登録しない。
- JenkinsはローカルPCのDocker Desktopで稼働させ、外部公開しない。
- JenkinsからEC2へSSHし、EC2上でGit更新とDocker Compose操作を行う。
- 破壊的操作は通常デプロイから分離し、確認パラメータを必須にする。
