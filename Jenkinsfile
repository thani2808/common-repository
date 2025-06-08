pipeline {
    agent any

    parameters {
        choice(name: 'APP_TYPE', choices: ['springboot', 'nginx'], description: 'App to deploy')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to use')
    }

    environment {
        GIT_REPO = 'git@github.com:your-username/bastion-apps.git'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: "${params.BRANCH}", url: "${env.GIT_REPO}"
            }
        }

        stage('Deploy in Parallel') {
            parallel {
                stage('SpringBoot') {
                    when { expression { params.APP_TYPE == 'springboot' } }
                    steps {
                        dir('springboot') {
                            sh 'docker build -t springboot-app .'
                            sh 'docker run -d -p 9000:9000 springboot-app'
                        }
                    }
                }
                stage('Nginx') {
                    when { expression { params.APP_TYPE == 'nginx' } }
                    steps {
                        dir('nginx') {
                            sh 'docker build -t nginx-app .'
                            sh 'docker run -d -p 8080:80 nginx-app'
                        }
                    }
                }
            }
        }
    }
}
