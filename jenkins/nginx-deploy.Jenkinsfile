pipeline {
  agent any
  options { disableConcurrentBuilds() }
  parameters {
    choice(name: 'ACTION', choices: ['APPLY', 'TEST_ONLY', 'RELOAD_ONLY'], description: 'Nginx操作 (APPLY: 設定転送・構文テスト・反映, TEST_ONLY: 構文テストのみ, RELOAD_ONLY: 反映のみ)')
    string(name: 'EC2_HOST', defaultValue: '', description: '対象EC2ホスト名またはIPアドレス')
    string(name: 'CONFIG_FILE_NAME', defaultValue: 'kykp.net.conf', description: '配置先Nginx設定ファイル名 (/etc/nginx/conf.d/ 配下)')
  }
  stages {
    stage('Validate') {
      steps {
        script {
          if (!(params.EC2_HOST ==~ /[A-Za-z0-9.-]+/)) error('EC2_HOSTパラメータが不正です')
          if (!(params.CONFIG_FILE_NAME ==~ /[A-Za-z0-9._-]+/)) error('CONFIG_FILE_NAMEパラメータが不正です')
        }
      }
    }
    stage('Deploy Nginx Config') {
      steps {
        sshagent(credentials: ['ec2-key-1']) {
          sh '''
            set -eu
            ACTION=$(echo "$ACTION" | tr '[:lower:]' '[:upper:]')
            if [ "$ACTION" = "APPLY" ]; then
              scp -o BatchMode=yes -o StrictHostKeyChecking=accept-new jenkins/nginx.conf ec2-user@"$EC2_HOST":/tmp/"$CONFIG_FILE_NAME"
            fi
            ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new ec2-user@"$EC2_HOST" \
              "ACTION='$ACTION' CONFIG_FILE_NAME='$CONFIG_FILE_NAME' sh -s" <<'REMOTE'
            set -eu
            if [ "$ACTION" = "APPLY" ]; then
              sudo install -m 0644 /tmp/"$CONFIG_FILE_NAME" /etc/nginx/conf.d/"$CONFIG_FILE_NAME"
              rm -f /tmp/"$CONFIG_FILE_NAME"
              sudo nginx -t
              sudo systemctl reload nginx
              echo "Nginx設定の配置、構文テスト、リロードが正常に完了しました。"
            elif [ "$ACTION" = "TEST_ONLY" ]; then
              sudo nginx -t
            elif [ "$ACTION" = "RELOAD_ONLY" ]; then
              sudo nginx -t
              sudo systemctl reload nginx
              echo "Nginxのリロードが正常に完了しました。"
            fi
REMOTE
          '''
        }
      }
    }
  }
}
