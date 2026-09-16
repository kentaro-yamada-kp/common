# 10.projects 共通ドキュメント

複数のWebアプリを同じAWSアカウント、原則1台のEC2で運用するための共通手順です。各プロジェクト固有の設定は各プロジェクトの `docs` を参照してください。

- [AWS共通運用](AWS共通運用.md)
- [一時資料：AWSプロファイル移行](tmp-AWSプロファイル移行.md)
- [Docker Desktop上のJenkins運用](Jenkins運用.md)
- [ローカルJenkins導入手順](Jenkins導入手順.md)
- [Docker Desktop・WSL導入手順](<Docker Desktop・WSL導入手順.md>)

AWSアクセスキーはEC2へ配置せず、必要なAWS APIはEC2のIAMロールで呼び出します。JenkinsはSSH経由で対象プロジェクトのデプロイ操作を実行します。
