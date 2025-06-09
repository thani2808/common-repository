pipeline {
    agent any

    parameters {
        choice(name: 'APP_TYPE', choices: ['springboot', 'nginx'], description: 'Type of app to deploy')
        choice(name: 'REPO_NAME', choices: [
            'hello-world-bastion',
            'dan-p81-bastion'
        ], description: 'Choose repository from thani2808')
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    def dockerhubUsername = "thanigai2808"
                    def bastionIp = "225.225.225.225"
                    def bastionUser = "ubuntu"
                    def defaultEnv = "dev"

                    def portMap = [dev: '9004', staging: '9005', prod: '9006']
                    def dockerPort = params.APP_TYPE == 'nginx' ? '80' : portMap[defaultEnv]
                    def hostPort = portMap[defaultEnv]

                    env.DOCKERHUB_USERNAME = dockerhubUsername
                    env.BASTION_IP = bastionIp
                    env.BASTION_USER = bastionUser
                    env.DEFAULT_ENV = defaultEnv
                    env.IMAGE_NAME = "${params.APP_TYPE}-bastion-app"
                    env.CONTAINER_NAME = "${params.APP_TYPE}-bastion-container"
                    env.DOCKER_PORT = dockerPort
                    env.HOST_PORT = hostPort
                    env.DOCKERHUB_REPO = "${dockerhubUsername}/${env.IMAGE_NAME}"
                    env.REPO_URL = "git@github.com:thani2808/${params.REPO_NAME}.git"
                }
            }
        }

        stage('Print Config') {
            steps {
                script {
                    echo "App Type      : ${params.APP_TYPE}"
                    echo "Repo Name     : ${params.REPO_NAME}"
                    echo "Bastion IP    : ${env.BASTION_IP}"
                    echo "Docker Repo   : ${env.DOCKERHUB_REPO}"
                    echo "Container     : ${env.CONTAINER_NAME}"
                    echo "Port Mapping  : ${env.HOST_PORT}:${env.DOCKER_PORT}"
                }
            }
        }

        stage('Clone the Repo') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/feature']],
                    userRemoteConfigs: [[
                        url: "${env.REPO_URL}",
                        credentialsId: 'private-key-jenkins'
                    ]]
                ])
            }
        }

        stage('Build App') {
            when { expression { return params.APP_TYPE == 'springboot' } }
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    if (!fileExists('Dockerfile')) {
                        error "❌ Dockerfile not found!"
                    }
                    sh "docker build -t ${env.IMAGE_NAME} ."
                }
            }
        }

        stage('Tag & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh """
                        echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                        docker tag ${env.IMAGE_NAME} ${env.DOCKERHUB_REPO}:${env.DEFAULT_ENV}
                        docker push ${env.DOCKERHUB_REPO}:${env.DEFAULT_ENV}
                        docker logout
                    """
                }
            }
        }

        stage('Deploy to Bastion') {
            steps {
                echo "🚀 Deploying container on Bastion..."
                withCredentials([sshUserPrivateKey(credentialsId: 'testing', keyFileVariable: 'keyf', usernameVariable: 'username')]) {
                    sh """
                        ssh-keyscan -H ${env.BASTION_IP} >> ~/.ssh/known_hosts
                        ssh -i ${keyf} ${env.BASTION_USER}@${env.BASTION_IP} << EOF
docker stop ${env.CONTAINER_NAME} || true
docker rm ${env.CONTAINER_NAME} || true
docker rmi ${env.DOCKERHUB_REPO}:${env.DEFAULT_ENV} || true
docker pull ${env.DOCKERHUB_REPO}:${env.DEFAULT_ENV}
docker run -d --name ${env.CONTAINER_NAME} -p ${env.HOST_PORT}:${env.HOST_PORT} ${env.DOCKERHUB_REPO}:${env.DEFAULT_ENV} java -jar app.jar --server.port=${env.HOST_PORT}
EOF
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                echo "🩺 Health checking app on Bastion..."
                withCredentials([sshUserPrivateKey(credentialsId: 'testing', keyFileVariable: 'keyf', usernameVariable: 'username')]) {
                    sh """
                        ssh -i ${keyf} ${env.BASTION_USER}@${env.BASTION_IP} << 'EOF'
set -x
retries=10
for i in \$(seq 1 \$retries); do
  RESPONSE_CODE=\$(curl -o /dev/null -s -w "%{http_code}" http://localhost:${env.HOST_PORT})
  if [[ "\$RESPONSE_CODE" == "200" ]]; then
    echo "✅ App is up!"
    exit 0
  else
    echo "Retry \$i/\$retries - Not ready"
    sleep 5
  fi
done
echo "❌ App failed to start"
exit 1
EOF
                    """
                }
            }
        }

        stage('Success') {
            steps {
                echo "🎉 Deployment of ${params.APP_TYPE} from ${params.REPO_NAME} succeeded!"
            }
        }
    }

    post {
        failure {
            echo '🚨 Pipeline failed!'
        }
        always {
            echo '📋 Pipeline finished.'
        }
    }
}
