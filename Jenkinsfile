ipeline {
    agent any

    environment {
        DOCKERHUB_REGISTRY = 'docker.io'
        COMMIT_TAG = sh(script: 'git describe --tags --abbrev=0', returnStdout: true).trim()
        USER_EMAIL = 'aleksandr_podkop@mail.ru'
        USER_NAME = 'Lepisok'
        BUILD_REASON = ""
    }

    stages {
        stage('Determine Build Reason') {
            steps {
                script {
                    if (currentBuild.rawBuild.getCause(com.github.events.TagCause)) {
                        env.BUILD_REASON = "Tag"
                    } else {
                        env.BUILD_REASON = "Commit"
                    }
                }
            }
        }

        stage('Check for Git Tag') {
            when {
                expression { env.COMMIT_TAG != null }
            }
            steps {
                script {
                    echo "Found Git Tag: ${env.COMMIT_TAG}"
                }
            }
        }

        stage('Check COMMIT_TAG') {
            steps {
                echo "Value of COMMIT_TAG: ${COMMIT_TAG}"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Используем извлеченный тег для сборки Docker-образа
                    sh "docker build -t lepisok/webserver:${env.DOCKER_TAG} ."
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-credentials') {
                        sh "docker info" // Add this line for debugging
                        sh "docker push lepisok/webserver:${env.DOCKER_TAG}"
                    }
                }
            }
        }

        // Остальные этапы...

        stage('Redeploy Kubernetes Deployment') {
            when {
                expression { env.COMMIT_TAG != null }
            }
            steps {
                script {
                    // Apply the updated Helm chart to your Kubernetes cluster
                    sh "helm upgrade nginx test_deploy/nginx"
                }
            }
        }
    }
}