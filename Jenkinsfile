pipeline {
    agent any

    parameters {
        choice(name: 'APP_TYPE', choices: ['springboot', 'nginx'], description: 'Type of app to deploy')
        choice(name: 'REPO_NAME', choices: [
            'hello-world-bastion',
            'dan-p81-bastion'
        ], description: 'Choose repository from thani2808')
    }

    environment {
        DOCKERHUB_USERNAME = 'thanigai2808'
        HOST_PORT = '9004'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    def portMap = [springboot: '9004', nginx: '80']
                    def dockerPort = portMap[params.APP_TYPE]

                    env.IMAGE_NAME = "${params.APP_TYPE}-local-app"
                    env.CONTAINER_NAME = "${params.APP_TYPE}-local-container"
                    env.DOCKER_PORT = dockerPort
                    env.DOCKERHUB_REPO = "${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}"
                    env.REPO_URL = "git@github.com:thani2808/${params.REPO_NAME}.git"
                }
            }
        }

        stage('Print Config') {
            steps {
                script {
                    echo "App Type      : ${params.APP_TYPE}"
                    echo "Repo Name     : ${params.REPO_NAME}"
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

        stage('Run Locally') {
            steps {
                script {
                    sh """
                        docker stop ${env.CONTAINER_NAME} || true
                        docker rm ${env.CONTAINER_NAME} || true
                        docker run -d --name ${env.CONTAINER_NAME} -p ${env.HOST_PORT}:${env.DOCKER_PORT} ${env.IMAGE_NAME} ${params.APP_TYPE == 'springboot' ? 'java -jar app.jar --server.port=' + env.HOST_PORT : ''}
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    sh """
                        retries=10
                        for i in \$(seq 1 \$retries); do
                          CODE=\$(curl -o /dev/null -s -w "%{http_code}" http://localhost:${env.HOST_PORT})
                          if [[ "\$CODE" == "200" ]]; then
                            echo "✅ App is up!"
                            exit 0
                          else
                            echo "⏳ Waiting for app... (\$i/\$retries)"
                            sleep 5
                          fi
                        done
                        echo "❌ App failed to start"
                        exit 1
                    """
                }
            }
        }

        stage('Success') {
            steps {
                echo "🎉 Local deployment of ${params.APP_TYPE} from ${params.REPO_NAME} succeeded!"
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
