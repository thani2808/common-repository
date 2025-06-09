pipeline {
    agent any

    parameters {
        choice(name: 'APP_TYPE', choices: ['springboot', 'nginx'], description: 'App to deploy')

        // Active Choices Parameter for GitHub repos
        [$class: 'CascadeChoiceParameter',
         choiceType: 'PT_SINGLE_SELECT',
         filterLength: 1,
         name: 'REPO_NAME',
         description: 'Choose the repository from thani2808',
         referencedParameters: '',
         script: [
            $class: 'GroovyScript',
            script: [
                sandbox: false,
                script: '''
                    def githubUser = "thani2808"
                    def repos = []
                    def conn = new URL("https://api.github.com/users/${githubUser}/repos").openConnection()
                    conn.setRequestProperty("User-Agent", "jenkins")
                    def response = new groovy.json.JsonSlurper().parse(conn.inputStream)
                    response.each {
                        repos << it.name
                    }
                    return repos
                '''
            ]
        ]]
    }

    environment {
        GIT_CREDENTIALS_ID = 'private-key-jenkins'
    }

    stages {
        stage('Clone Repository') {
            steps {
                script {
                    def repoURL = "git@github.com:thani2808/${params.REPO_NAME}.git"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/main']],
                        userRemoteConfigs: [[
                            url: repoURL,
                            credentialsId: env.GIT_CREDENTIALS_ID
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
}
