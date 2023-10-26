pipeline {
    agent any

    environment {
    DOCKERHUB_REGISTRY = 'docker.io'
    COMMIT_TAG = sh(script: 'git describe --tags --abbrev=0', returnStdout: true).trim()
    USER_EMAIL = 'aleksandr_podkop@mail.ru'
    USER_NAME = 'Lepisok'
    // Add other environment variables here if needed
}

    stages {
        stage('Extract Tag from Commit') {
            steps {
                script {
                    // Извлекаем тег из коммита
                    def extractedTag = sh(script: 'git describe --tags --abbrev=0', returnStdout: true).trim()

                    echo "Извлеченный тег: ${extractedTag}"

                    if (extractedTag =~ /\d+\.\d+/) {
                        env.DOCKER_TAG = extractedTag
                    } else {
                        error "Invalid tag format: ${extractedTag}"
                    }
                }
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
                            cat values.yaml | sed -e "s/imageTag: v/imageTag: \${COMMIT_TAG}/" > values.tmp.yaml
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
                    dir("test_deploy") {
                            withCredentials([file(credentialsId: 'github', variable: 'SSH_KEY')]) {
                            sh '''
                                eval $(ssh-agent -s)
                                git add .
                                git commit -m "Build #\${BUILD_NUMBER}"
                                git push git@github.com:Lepisok/test_deploy.git
                            '''
                        }
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

        stage('Redeploy Kubernetes Deployment') {
            steps {
                script {
                    // Update the Docker image version in your Helm chart values.yaml file
                    sh "sed -i 's/imageTag: v/imageTag: v2/' test_deploy/nginx/values.yaml"
                    
                    // Apply the updated Helm chart to your Kubernetes cluster
                    sh "helm upgrade nginx test_deploy/nginx"
                }
            }
        }
    
    }
}