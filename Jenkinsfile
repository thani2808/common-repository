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
                            // Build JAR
                            sh './mvnw clean package -DskipTests || mvn clean package -DskipTests'
                            
                            // Build Docker image
                            sh 'docker build -t springboot-app .'

                            // Remove old container
                            sh 'docker rm -f springboot-container || true'

                            // Run app on host port 9010, container port 9010 (make sure Spring Boot uses 9010)
                            sh 'docker run -d -p 9010:9010 --name springboot-container springboot-app'

                            // Optional: wait for container to start and log a bit
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
                            // Build Docker image
                            sh 'docker build -t nginx-app .'

                            // Remove old container
                            sh 'docker rm -f nginx-container || true'

                            // Run app on port 8001
                            sh 'docker run -d -p 8002:80 --name nginx-container nginx-app'
                        }
                    }
                }
            }
        }
    }
}
