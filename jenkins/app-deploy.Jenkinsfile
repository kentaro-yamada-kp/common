pipeline {
  agent any
  options { disableConcurrentBuilds() }
  parameters {
    choice(name: 'ACTION', choices: ['DEPLOY', 'RESTART', 'STOP'], description: 'アプリ操作')
    choice(name: 'PROJECT', choices: ['crane-rank', 'generic-matching'], description: 'プロジェクト名')
    choice(name: 'BRANCH', choices: ['main', 'develop'], description: 'デプロイブランチ')
    string(name: 'EC2_HOST', defaultValue: 'kykp.net', description: '対象EC2')
  }
  stages {
    stage('Validate') { steps { script {
      if (!(params.PROJECT ==~ /[A-Za-z0-9._-]+/) || !(params.BRANCH ==~ /[A-Za-z0-9._\/-]+/) || !(params.EC2_HOST ==~ /[A-Za-z0-9.-]+/)) error('パラメータが不正です')
    } } }
    stage('Deploy or operate') {
      steps {
        sshagent(credentials: ['ec2-key-1']) {
          script {
            def portMap = ['crane-rank': '3001', 'generic-matching': '3000']
            env.APP_PORT_VAL = portMap[params.PROJECT] ?: '3000'
            env.REPOSITORY_URL_VAL = "https://github.com/kentaro-yamada-kp/${params.PROJECT}.git"
            env.ENV_CREDENTIAL_ID_VAL = "${params.PROJECT}-env-production"
            env.HEALTH_CHECK_PATH_VAL = '/api/health'
            env.COMPOSE_FILE_VAL = 'docker-compose.production.yml'
          }
          withCredentials([file(credentialsId: env.ENV_CREDENTIAL_ID_VAL, variable: 'PRODUCTION_ENV')]) {
            sh '''
              set -eu
              ACTION=$(echo "$ACTION" | tr '[:lower:]' '[:upper:]')
              REMOTE_DIR="/home/ec2-user/$PROJECT"
              REPOSITORY_URL="$REPOSITORY_URL_VAL"
              APP_PORT="$APP_PORT_VAL"
              HEALTH_CHECK_PATH="$HEALTH_CHECK_PATH_VAL"
              COMPOSE_FILE="$COMPOSE_FILE_VAL"
              if [ "$ACTION" = DEPLOY ]; then
                scp -o BatchMode=yes -o StrictHostKeyChecking=accept-new "$PRODUCTION_ENV" ec2-user@"$EC2_HOST":/tmp/"$PROJECT".env
              fi
              ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new ec2-user@"$EC2_HOST" \
                "ACTION='$ACTION' PROJECT='$PROJECT' BRANCH='$BRANCH' REPOSITORY_URL='$REPOSITORY_URL' REMOTE_DIR='$REMOTE_DIR' APP_PORT='$APP_PORT' HEALTH_CHECK_PATH='$HEALTH_CHECK_PATH' COMPOSE_FILE='$COMPOSE_FILE' sh -s" <<'REMOTE'
              set -eu
              TARGET_COMPOSE="$COMPOSE_FILE"
              if [ "$ACTION" = DEPLOY ]; then
                if [ ! -d "$REMOTE_DIR/.git" ]; then git clone --branch "$BRANCH" "$REPOSITORY_URL" "$REMOTE_DIR"; fi
                cd "$REMOTE_DIR"
                git fetch origin "$BRANCH"
                git checkout "$BRANCH"
                git pull --ff-only origin "$BRANCH"
                install -m 600 /tmp/"$PROJECT".env .env.production
                rm -f /tmp/"$PROJECT".env
                if [ ! -f "$TARGET_COMPOSE" ] && [ -f docker-compose.yml ]; then TARGET_COMPOSE="docker-compose.yml"; fi
                docker compose -f "$TARGET_COMPOSE" up -d --build --remove-orphans
                if [ -f jenkins/nginx.conf ]; then sudo install -m 0644 jenkins/nginx.conf /etc/nginx/conf.d/"$PROJECT".conf; sudo nginx -t; sudo systemctl reload nginx; fi
              elif [ "$ACTION" = RESTART ]; then
                cd "$REMOTE_DIR"
                if [ ! -f "$TARGET_COMPOSE" ] && [ -f docker-compose.yml ]; then TARGET_COMPOSE="docker-compose.yml"; fi
                docker compose -f "$TARGET_COMPOSE" up -d --force-recreate
              else
                cd "$REMOTE_DIR"
                if [ ! -f "$TARGET_COMPOSE" ] && [ -f docker-compose.yml ]; then TARGET_COMPOSE="docker-compose.yml"; fi
                docker compose -f "$TARGET_COMPOSE" stop
                exit 0
              fi
              for i in $(seq 1 15); do curl -fsS "http://127.0.0.1:$APP_PORT$HEALTH_CHECK_PATH" && exit 0; sleep 2; done
              docker compose -f "$TARGET_COMPOSE" logs --tail=200 >&2
              exit 1
REMOTE
            '''
          }
        }
      }
    }
  }
}
