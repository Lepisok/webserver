pipeline {
    agent any

    environment {
        DOCKERHUB_REGISTRY = 'docker.io'
        USER_EMAIL = 'aleksandr_podkop@mail.ru'
        USER_NAME = 'Lepisok'
    }

    stages {
        stage('GitHub Event Handling') {
            when {
                expression { currentBuild.changeSets[0].commitId }
            }
            steps {
                script {
                    def eventCause = currentBuild.rawBuild.getCause(com.cloudbees.jenkins.GitHubTagCause)
                    if (eventCause != null) {
                        env.GIT_TAG_BUILD = 'true'
                        env.COMMIT_TAG = eventCause.tag
                    } else {
                        env.COMMIT_TAG = ""
                    }
                }
            }
        }

        stage('Check COMMIT_TAG') {
            when {
                expression { env.GIT_TAG_BUILD == 'true' }
            }
            steps {
                echo "Value of COMMIT_TAG: ${env.COMMIT_TAG}"
            }
        }

        stage('Build Docker Image') {
            when {
                anyOf {
                    githubPush(); // Реагировать на коммиты в репозиторий
                    expression { env.COMMIT_TAG != null } // Сборка по git-тегу
                }
            }
            steps {
                script {
                    sh "docker build -t lepisok/webserver:${env.DOCKER_TAG} ."
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            when {
                anyOf {
                    githubPush(); // Реагировать на коммиты в репозиторий
                    expression { env.COMMIT_TAG != null } // Сборка по git-тегу
                }
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-credentials') {
                        sh "docker info"
                        sh "docker push lepisok/webserver:${env.DOCKER_TAG}"
                    }
                }
            }
        }

        // Остальные этапы оставьте без изменений

        stage('Redeploy Kubernetes Deployment') {
            when {
                expression { env.COMMIT_TAG != null } // Деплой только при сборке по git-тегу
            }
            steps {
                script {
                    sh "helm upgrade nginx test_deploy/nginx"
                }
            }
        }
    }
}