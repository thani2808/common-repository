pipeline {
    agent any

    parameters {
        choice(name: 'APP_TYPE', choices: ['springboot', 'nginx'], description: 'App to deploy')
        string(name: 'BRANCH', defaultValue: 'feature', description: 'Git branch to use')
    }

    environment {
        GIT_REPO = 'git@github.com:thani2808/common-repository.git'
        GIT_CREDENTIALS_ID = 'private-key-jenkins'
    }

    stages {
        stage('Clone Repository') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${params.BRANCH}"]],
                    userRemoteConfigs: [[
                        url: env.GIT_REPO,
                        credentialsId: env.GIT_CREDENTIALS_ID
                    ]]
                ])
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
                            sh 'mvn clean package -DskipTests'
                            sh 'docker build -t springboot-app .'
                            // Remove any existing container before running
                            sh 'docker rm -f springboot-container || true'
                            sh 'docker run -d -p 9010:9010 --name springboot-container springboot-app'
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
                            // Remove any existing container before running
                            sh 'docker rm -f nginx-container || true'
                            sh 'docker run -d -p 8001:80 --name nginx-container nginx-app'
                        }
                    }
                }
            }
        }
    }
}
