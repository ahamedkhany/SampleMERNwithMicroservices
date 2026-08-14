pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'

        FRONTEND_REPO = 'streaming-frontend'
        HELLO_REPO = 'streaming-hello-service'
        PROFILE_REPO = 'streaming-profile-service'

        AWS_CREDENTIALS_ID = 'aws-ecr-credentials'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code from GitHub...'
                checkout scm
            }
        }

        stage('Get AWS Account ID') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: env.AWS_CREDENTIALS_ID,
                            usernameVariable: 'AWS_ACCESS_KEY_ID',
                            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {

                        env.AWS_ACCOUNT_ID = sh(
                            script: '''
                                aws sts get-caller-identity \
                                  --query Account \
                                  --output text \
                                  --no-cli-pager
                            ''',
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
        }

        stage('Build Frontend Image') {
            steps {
                echo 'Building Frontend Docker image...'

                sh '''
                    docker build \
                      -t ${FRONTEND_REPO}:${BUILD_NUMBER} \
                      ./frontend
                '''
            }
        }

        stage('Build Hello Service Image') {
            steps {
                echo 'Building Hello Service Docker image...'

                sh '''
                    docker build \
                      -t ${HELLO_REPO}:${BUILD_NUMBER} \
                      ./backend/helloService
                '''
            }
        }

        stage('Build Profile Service Image') {
            steps {
                echo 'Building Profile Service Docker image...'

                sh '''
                    docker build \
                      -t ${PROFILE_REPO}:${BUILD_NUMBER} \
                      ./backend/profileService
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.AWS_CREDENTIALS_ID,
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {

                    sh '''
                        echo "Logging in to Amazon ECR..."

                        aws ecr get-login-password \
                          --region ${AWS_REGION} |
                        docker login \
                          --username AWS \
                          --password-stdin ${ECR_REGISTRY}
                    '''
                }
            }
        }

        stage('Tag Images') {
            steps {
                echo 'Tagging Docker images for ECR...'

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
                echo 'Pushing Docker images to Amazon ECR...'

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
            echo 'CI PIPELINE SUCCESSFUL'
            echo 'All Docker images were pushed to Amazon ECR.'
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