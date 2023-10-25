pipeline {
    agent any

    environment {
    DOCKERHUB_REGISTRY = 'docker.io'
    // Add other environment variables here if needed
}

    stages {
        stage('Extract Tag from Commit') {
            steps {
                script {
                    // Извлекаем тег из коммита
                    env.DOCKER_TAG = sh(script: 'git describe --tags --abbrev=0', returnStdout: true).trim()

                    echo "Извлеченный тег: ${env.DOCKER_TAG}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Используем извлеченный тег для сборки Docker-образа
                    sh "docker build -t lepisok:${env.DOCKER_TAG} ."
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'lepisok', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD')]) {
                        sh "echo ${DOCKERHUB_PASSWORD} | docker login ${DOCKERHUB_REGISTRY} -u ${DOCKERHUB_USERNAME} --password-stdin"
                    }
                }
            }
        }

        stage('Get Helm Repository Contents') {
            steps {
                script {
                    if (fileExists('test_deploy')) {
                        dir('test_deploy') {
                            sh 'git pull origin main' // Update the repository
                        }
                    } else {
                        sh "git clone https://github.com/Lepisok/test_deploy"
                    }
                }
            }
        }

        stage('Update Chart Version') {
            steps {
                script {
                    dir("test_deploy") {
                        dir("nginx") {
                            sh """
                                cat Chart.yaml | sed -e "s/version:.*/version: \${COMMIT_TAG}/" > Chart.tmp.yaml
                                mv Chart.tmp.yaml Chart.yaml

                                cat Chart.yaml | sed -e "s/appVersion:.*/appVersion: \"\${COMMIT_TAG}\"/" > Chart.tmp.yaml
                                mv Chart.tmp.yaml Chart.yaml
                            """
                        }
                    }
                }
            }
        }

        stage('Update Image Tag') {
            steps {
                script {
                    dir("test_deploy/nginx") {
                        sh """
                            cat Chart.yaml | sed -e "s/version:.*/version: \${COMMIT_TAG}/" > Chart.tmp.yaml
                            mv Chart.tmp.yaml Chart.yaml
                        """
                    }
                }
            }
        }

        stage('Create Helm Archive') {
            steps {
                script {
                    dir("test_deploy") {
                        sh "helm package nginx -d chart"
                    }
                }
            }
        }

        stage('Update Helm Index') {
            steps {
                script {
                    dir("test_deploy/nginx/chart") {
                        sh "helm repo index ."
                    }
                }
            }
        }

        stage('Push to Git Repository') {
            steps {
                script {
                    dir("test_deploy/nginx") {
                        sh """
                            git add .
                            git config --global user.email "${USER_EMAIL}"
                            git config --global user.name "${USER_NAME}"
                            git commit -m "Build #\${BUILD_NUMBER}"
                            git push origin -f
                        """
                    }
                }
            }
        }

        stage('Cleanout') {
            steps {
                script {
                    dir("nginx") {
                        deleteDir()
                    }
                }
            }
        }

        stage('Update Previous Commit Tag') {
            steps {
                echo "##jenkins[setParameter name='PREV_COMMIT_TAG' value='${COMMIT_TAG}']"
            }
        }
    }
}