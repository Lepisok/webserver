pipeline {
    agent any

    environment {
    DOCKERHUB_REGISTRY = 'docker.io'
    USER_EMAIL = 'aleksandr_podkop@mail.ru'
    USER_NAME = 'Lepisok'
    // Add other environment variables here if needed
}


stages {
    stage('Extract Tag from Event') {
        when {
            expression { currentBuild.rawBuild.getCause(com.cloudbees.jenkins.GitHubPushCause) != null }
        }
        steps {
            script {
                def eventCause = currentBuild.rawBuild.getCause(com.cloudbees.jenkins.GitHubPushCause)
                def eventInfo = eventCause.shortDescription

                echo "GitHub Event Info: ${eventInfo}"

                // Извлекаем тег из описания события (пример: "Push event to branch master at commit xxx")
                def extractedTag = eventInfo =~ /Push event to branch .+ at commit .+/

                if (extractedTag) {
                    env.DOCKER_TAG = extractedTag[0][0..12] // Пример: извлекаем первые 13 символов (длина формата тега)
                } else {
                    error "Failed to extract tag from event info"
                }
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
                    dir("test_deploy/nginx") {
                            sh """
                                cat Chart.yaml | sed -e "s/version:.*/version: \${COMMIT_TAG}/" > Chart.tmp.yaml
                                mv Chart.tmp.yaml Chart.yaml

                                cat Chart.yaml | sed -e "s/appVersion:.*/appVersion: \\"\${COMMIT_TAG}\\"/" > Chart.tmp.yaml
                                mv Chart.tmp.yaml Chart.yaml
                            """
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
            when {
                expression { currentBuild.rawBuild.getCause(com.cloudbees.jenkins.GitHubTagCause) != null }
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