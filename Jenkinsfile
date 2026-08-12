pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_PAGER = ''

        FRONTEND_REPO = 'streaming-frontend'
        HELLO_REPO = 'streaming-hello-service'
        PROFILE_REPO = 'streaming-profile-service'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Get AWS Account ID') {
            steps {
                script {
                    env.AWS_ACCOUNT_ID = sh(
                        script: 'aws sts get-caller-identity --query Account --output text --no-cli-pager',
                        returnStdout: true
                    ).trim()

                    env.ECR_REGISTRY =
                        "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"

                    env.FRONTEND_IMAGE =
                        "${env.ECR_REGISTRY}/${env.FRONTEND_REPO}"

                    env.HELLO_IMAGE =
                        "${env.ECR_REGISTRY}/${env.HELLO_REPO}"

                    env.PROFILE_IMAGE =
                        "${env.ECR_REGISTRY}/${env.PROFILE_REPO}"

                    echo "AWS Account ID: ${env.AWS_ACCOUNT_ID}"
                    echo "ECR Registry: ${env.ECR_REGISTRY}"
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    docker build \
                      -t ${FRONTEND_REPO}:${BUILD_NUMBER} \
                      ./frontend
                '''
            }
        }

        stage('Build Hello Service Image') {
            steps {
                sh '''
                    docker build \
                      -t ${HELLO_REPO}:${BUILD_NUMBER} \
                      ./backend/helloService
                '''
            }
        }

        stage('Build Profile Service Image') {
            steps {
                sh '''
                    docker build \
                      -t ${PROFILE_REPO}:${BUILD_NUMBER} \
                      ./backend/profileService
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region ${AWS_REGION} \
                      --no-cli-pager |
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Tag Images') {
            steps {
                sh '''
                    docker tag \
                      ${FRONTEND_REPO}:${BUILD_NUMBER} \
                      ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    docker tag \
                      ${HELLO_REPO}:${BUILD_NUMBER} \
                      ${HELLO_IMAGE}:${BUILD_NUMBER}

                    docker tag \
                      ${PROFILE_REPO}:${BUILD_NUMBER} \
                      ${PROFILE_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    docker push ${HELLO_IMAGE}:${BUILD_NUMBER}

                    docker push ${PROFILE_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }
    }

    post {

        success {
            echo '=============================================='
            echo 'CI PIPELINE COMPLETED SUCCESSFULLY'
            echo 'All three images have been pushed to ECR.'
            echo '=============================================='
        }

        failure {
            echo '=============================================='
            echo 'CI PIPELINE FAILED'
            echo 'Check the Jenkins Console Output.'
            echo '=============================================='
        }

        always {
            sh '''
                docker image prune -f || true
            '''
        }
    }
}