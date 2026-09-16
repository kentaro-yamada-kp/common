pipeline {
  agent any
  options { disableConcurrentBuilds() }
  parameters {
    choice(name: 'ACTION', choices: ['STATUS', 'START', 'STOP'], description: 'EC2の状態確認・起動・停止')
    string(name: 'TARGET_NAME', defaultValue: 'ec2-1', description: 'EC2のNameタグ')
    string(name: 'AWS_REGION', defaultValue: 'ap-northeast-1', description: 'AWSリージョン')
  }
  stages {
    stage('Validate') { steps { script { if (!(params.TARGET_NAME ==~ /[A-Za-z0-9._-]+/) || !(params.AWS_REGION ==~ /[a-z]{2}-[a-z]+-[0-9]/)) error('パラメータが不正です') } } }
    stage('Operate EC2') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''
            set -eu
            ACTION=$(echo "$ACTION" | tr '[:lower:]' '[:upper:]')
            ID=$(aws ec2 describe-instances --region "$AWS_REGION" --filters "Name=tag:Name,Values=$TARGET_NAME" "Name=instance-state-name,Values=pending,running,stopping,stopped" --query 'Reservations[0].Instances[0].InstanceId' --output text)
            test -n "$ID" && test "$ID" != None
            STATE=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$ID" --query 'Reservations[0].Instances[0].State.Name' --output text)
            echo "Instance=$ID State=$STATE"
            case "$ACTION:$STATE" in
              STATUS:*) exit 0 ;;
              START:running|START:pending) exit 0 ;;
              START:stopped) aws ec2 start-instances --region "$AWS_REGION" --instance-ids "$ID"; aws ec2 wait instance-running --region "$AWS_REGION" --instance-ids "$ID" ;;
              STOP:stopped|STOP:stopping) exit 0 ;;
              STOP:running) aws ec2 stop-instances --region "$AWS_REGION" --instance-ids "$ID"; aws ec2 wait instance-stopped --region "$AWS_REGION" --instance-ids "$ID" ;;
              *) echo "現在の状態では実行できません: $ACTION / $STATE" >&2; exit 1 ;;
            esac
          '''
        }
      }
    }
  }
}
