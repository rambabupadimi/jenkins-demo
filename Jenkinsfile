pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk   'JDK25'
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        PATH          = "/opt/homebrew/bin:/usr/local/bin:/Applications/Docker.app/Contents/Resources/bin:${env.PATH}"
        DOCKER_CONFIG = "${env.WORKSPACE}/.docker"
        IMAGE_NAME    = 'rambabupadimi/jenkins-demo'
        IMAGE_TAG     = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Check tools') {
            steps {
                sh '''
                    java -version
                    mvn -version
                    docker --version
                    kubectl version --client
                '''
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .'
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        mkdir -p "$DOCKER_CONFIG"
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f k8s/deployment.yaml
                    kubectl set image deployment/jenkins-demo jenkins-demo=${IMAGE_NAME}:${IMAGE_TAG}
                    kubectl rollout status deployment/jenkins-demo --timeout=120s
                '''
            }
        }
    }

    post {
        success { echo 'Pipeline succeeded' }
        failure { echo 'Pipeline failed, check the stage logs' }
    }
}