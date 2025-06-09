pipeline {
    agent any

    stages {
        stage('Initialize') {
            steps {
                script {
                    // Define job parameters here so they appear in UI
                    properties([
                        parameters([
                            choice(
                                name: 'APP_TYPE',
                                choices: ['springboot', 'nginx'],
                                description: 'App to deploy'
                            ),
                            choice(
                                name: 'REPO_NAME',
                                choices: [
                                    'hello-world-bastion',
                                    'dan-p81-bastion'
                                ],
                                description: 'Choose repository from thani2808'
                            )
                        ])
                    ])
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
                    if (!params.REPO_NAME) {
                        error("❌ Repository name not selected. Please choose a valid repository.")
                    }
                    def repoURL = "git@github.com:thani2808/${params.REPO_NAME}.git"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/feature']],
                        userRemoteConfigs: [[
                            url: repoURL,
                            credentialsId: 'private-key-jenkins'
                        ]]
                    ])
                }
            }
        }

        stage('Deploy in Parallel') {
            parallel {
                stage('SpringBoot') {
                    when {
                        expression { params.APP_TYPE == 'springboot' }
                    }
                    steps {
                        dir('springboot') {
                            sh './mvnw clean package -DskipTests || mvn clean package -DskipTests'
                            sh 'docker build -t springboot-app .'
                            sh 'docker rm -f springboot-container || true'
                            sh 'docker run -d -p 9010:9010 --name springboot-container springboot-app'
                            sh 'sleep 5 && docker logs springboot-container --tail 10'
                        }
                    }
                }

                stage('Nginx') {
                    when {
                        expression { params.APP_TYPE == 'nginx' }
                    }
                    steps {
                        dir('nginx') {
                            sh 'docker build -t nginx-app .'
                            sh 'docker rm -f nginx-container || true'
                            sh 'docker run -d -p 8002:80 --name nginx-container nginx-app'
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            echo '❌ Build failed. Please check logs above.'
        }
    }
}
