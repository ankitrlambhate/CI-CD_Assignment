// Jenkinsfile for the StreamingApp microservices (auth, streaming, admin, chat, frontend).
// Uses the repo's own docker-compose.yml to build every image, then pushes each one to its ECR repo:
//   streamingapp-auth, streamingapp-streaming, streamingapp-admin, streamingapp-chat, streamingapp-frontend
//
// Jenkins credentials needed (kind: Secret text):
//   ankit-aws-access-key-id
//   ankit-aws-secret-access-key
pipeline {
    agent any
 
    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 60, unit: 'MINUTES')
    }
 
    triggers {
        githubPush()
    }
 
    environment {
        AWS_REGION            = 'ap-south-1'
        SERVICES              = 'auth streaming admin chat frontend'
        SNS_TOPIC_ARN         = 'arn:aws:sns:ap-south-1:235063678427:streamingapp-alerts'                       // optional (bonus ChatOps)
        IMAGE_TAG             = "${env.BUILD_NUMBER}"
 
        // Makes compose name images "streamingapp-<service>"
        COMPOSE_PROJECT_NAME  = 'streamingapp'
        AWS_ACCESS_KEY_ID     = credentials('ankit-aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('ankit-aws-secret-access-key')
        AWS_DEFAULT_REGION    = 'ap-south-1'
        PATH                  = "${env.WORKSPACE}/bin:${env.PATH}"
    }
 
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'ls -la'
            }
        }
 
        stage('Check Tools') {
            steps {
                sh '''
                  mkdir -p "$WORKSPACE/bin"
 
                  docker --version || { echo "Docker is not available to Jenkins on this server."; exit 1; }
                  docker info >/dev/null 2>&1 || { echo "Jenkins cannot talk to the Docker daemon."; exit 1; }
                  docker compose version || { echo "'docker compose' plugin missing on this server."; exit 1; }
 
                  if ! command -v aws >/dev/null 2>&1; then
                    curl -sSL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip
                    python3 -m zipfile -e /tmp/awscliv2.zip /tmp/awscli-src
                    chmod +x /tmp/awscli-src/aws/install /tmp/awscli-src/aws/dist/aws
                    /tmp/awscli-src/aws/install -i "$WORKSPACE/awscli" -b "$WORKSPACE/bin" --update
                  fi
                  aws --version
                '''
            }
        }
 
        stage('Verify AWS Account') {
            steps {
                script {
                    env.ACCOUNT_ID = sh(returnStdout: true,
                        script: 'aws sts get-caller-identity --query Account --output text').trim()
                    env.REGISTRY = "${env.ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                    echo "Using AWS account ${env.ACCOUNT_ID}"   // must be YOUR account number
                }
            }
        }
 
        stage('Ensure ECR Repos') {
            steps {
                sh '''
                  for s in $SERVICES; do
                    aws ecr describe-repositories --repository-names streamingapp-$s --region $AWS_REGION >/dev/null 2>&1 \
                      || aws ecr create-repository --repository-name streamingapp-$s --region $AWS_REGION \
                           --image-scanning-configuration scanOnPush=true
                  done
                '''
            }
        }
 
        stage('Build Images') {
            steps {
                sh 'docker compose build'
            }
        }
 
        stage('Tag and Push to ECR') {
            steps {
                sh '''
                  aws ecr get-login-password --region $AWS_REGION \
                    | docker login --username AWS --password-stdin $REGISTRY
 
                  for s in $SERVICES; do
                    SRC="streamingapp-$s"
                    docker image inspect "$SRC" >/dev/null 2>&1 \
                      || { echo "Image $SRC not found. Check the service name in docker-compose.yml."; exit 1; }
                    docker tag "$SRC" "$REGISTRY/streamingapp-$s:$IMAGE_TAG"
                    docker tag "$SRC" "$REGISTRY/streamingapp-$s:latest"
                    docker push "$REGISTRY/streamingapp-$s:$IMAGE_TAG"
                    docker push "$REGISTRY/streamingapp-$s:latest"
                  done
                '''
            }
        }
    }
 
    post {
        success {
            script { notify('SUCCESS', "Build #${env.BUILD_NUMBER} pushed all images (${env.JOB_NAME})") }
        }
        failure {
            script { notify('FAILED', "Build #${env.BUILD_NUMBER} failed (${env.JOB_NAME}) ${env.BUILD_URL}") }
        }
        always {
            sh 'docker logout $REGISTRY || true; docker image prune -f || true'
        }
    }
}
 
def notify(String status, String msg) {
    if (env.SNS_TOPIC_ARN?.trim()) {
        sh """aws sns publish --region ${env.AWS_REGION} \
              --topic-arn ${env.SNS_TOPIC_ARN} \
              --subject 'StreamingApp CI/CD: ${status}' \
              --message '${msg}'"""
    }
}
