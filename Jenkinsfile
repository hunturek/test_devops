pipeline {
    agent any
    environment {
        // Общие переменные
        DOCKER_REGISTRY = 'docker.io' // Укажите ваш Docker registry
        
        // Бэкенд переменные
        BACKEND_REPO = 'https://github.com/Emilien-mipt/titanic-webapp.git'
        BACKEND_BRANCH = 'main'
        BACKEND_DOCKERFILE = 'Dockerfile' // или просто Dockerfile, если он так называется
        BACKEND_IMAGE_NAME = 'hunturek/titanic-model'
        
        // Фронтенд переменные
        FRONTEND_REPO = 'https://github.com/hunturek/test_frontend.git'
        FRONTEND_BRANCH = 'develop'
        FRONTEND_DOCKERFILE = 'Dockerfile' // или просто Dockerfile, если он так называется
        FRONTEND_IMAGE_NAME = 'hunturek/titanic-predictor'
        
        // DevOps переменные
        DEVOPS_REPO = 'https://github.com/hunturek/test_devops.git'
        DEVOPS_BRANCH = 'develop'
    }
    
    stages {
        stage('Prepare') {
            steps {
                script {
                    // Очистка workspace перед началом
                    cleanWs()
                }
            }
        }
        
        stage('Checkout and Build Backend') {
            steps {
                script {
                    // Клонируем бэкенд репозиторий
                    git branch: env.BACKEND_BRANCH, url: env.BACKEND_REPO
                    
                    // Собираем Docker образ для бэкенда
                    docker.build("${env.DOCKER_REGISTRY}/${env.BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}", "-f ${env.BACKEND_DOCKERFILE} --build-arg PIP_EXTRA_INDEX_URL=https://pypi.org/project/titanic-model/ .")
                    
                    // Пушим образ в registry (если нужно)
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS} ${env.DOCKER_REGISTRY}"
                        bat "docker push ${env.DOCKER_REGISTRY}/${env.BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    }
                }
            }
        }
        
        stage('Checkout and Build Frontend') {
            steps {
                script {
                    // Клонируем фронтенд репозиторий
                    git branch: env.FRONTEND_BRANCH, url: env.FRONTEND_REPO
                    
                    // Собираем Docker образ для фронтенда
                    docker.build("${env.DOCKER_REGISTRY}/${env.FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}", "-f ${env.FRONTEND_DOCKERFILE} .")
                    
                    // Пушим образ в registry (если нужно)
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat "docker push ${env.DOCKER_REGISTRY}/${env.FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Checkout DevOps Repo') {
            steps {
                dir('devops-repo') {  // Клонируем репозиторий в подкаталог devops-repo
                    git branch: env.DEVOPS_BRANCH, url: env.DEVOPS_REPO
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'k8s-jenkins-token', variable: 'K8S_TOKEN')]) {
                        
                        bat """
                            kubectl config set-cluster k8s-cluster --server=https://127.0.0.1:59580
                            kubectl config set-credentials jenkins --token=%K8S_TOKEN%
                            kubectl config set-context jenkins-context --cluster=k8s-cluster --user=jenkins
                            kubectl config use-context jenkins-context
                        """
                        
                        bat "kubectl --insecure-skip-tls-verify=true apply -f ${env.WORKSPACE}/devops-repo/backend/api-deployment.yaml --validate=false"
                        bat "kubectl --insecure-skip-tls-verify=true apply -f ${env.WORKSPACE}/devops-repo/backend/api-service.yaml --validate=false"
                        
                        bat "kubectl --insecure-skip-tls-verify=true apply -f ${env.WORKSPACE}/devops-repo/frontend/ui-deployment.yaml --validate=false"
                        bat "kubectl --insecure-skip-tls-verify=true apply -f ${env.WORKSPACE}/devops-repo/frontend/ui-service.yaml --validate=false"

                        def apiContainer = bat(
                            script: 'kubectl --insecure-skip-tls-verify=true get deployment titanic-api -o jsonpath="{.spec.template.spec.containers[0].name}"',
                            returnStdout: true
                        ).trim()
                        
                        def uiContainer = bat(
                            script: 'kubectl --insecure-skip-tls-verify=true get deployment titanic-ui -o jsonpath="{.spec.template.spec.containers[0].name}"',
                            returnStdout: true
                        ).trim()
                        
                        bat """
                            kubectl --insecure-skip-tls-verify=true set image deployment/titanic-api ${apiContainer}=${env.DOCKER_REGISTRY}/${env.BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}
                            kubectl --insecure-skip-tls-verify=true set image deployment/titanic-ui ${uiContainer}=${env.DOCKER_REGISTRY}/${env.FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}