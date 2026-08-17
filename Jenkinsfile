pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'

        FRONTEND_REPO = 'streaming-frontend'
        HELLO_REPO    = 'streaming-hello-service'
        PROFILE_REPO  = 'streaming-profile-service'

        AWS_CREDENTIALS_ID = 'aws-ecr-credentials'

        EKS_CLUSTER_NAME = 'container-orch-cluster'

        K8S_MANIFEST_PATH = 'infrastructure/eks/manifests'
    }

    stages {

        // ============================================================
        // CI - SOURCE CODE
        // ============================================================

        stage('Checkout') {
            steps {
                echo 'Checking out source code from GitHub...'

                checkout scm
            }
        }

        stage('Generate Image Version') {
            steps {
                script {

                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    /*
                     * Dynamic and traceable image version.
                     *
                     * Example:
                     * build-28-a1b2c3d
                     */
                    env.IMAGE_TAG =
                        "build-${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"

                    echo "=============================================="
                    echo "IMAGE VERSION"
                    echo "=============================================="
                    echo "Jenkins Build : ${env.BUILD_NUMBER}"
                    echo "Git Commit    : ${env.GIT_COMMIT_SHORT}"
                    echo "Image Tag     : ${env.IMAGE_TAG}"
                    echo "=============================================="
                }
            }
        }

        // ============================================================
        // CI - AWS / ECR INFORMATION
        // ============================================================

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

                        echo "AWS Account ID : ${env.AWS_ACCOUNT_ID}"
                        echo "ECR Registry   : ${env.ECR_REGISTRY}"
                    }
                }
            }
        }

        // ============================================================
        // CI - DOCKER BUILD
        // ============================================================

        stage('Build Frontend Image') {
            steps {
                echo 'Building Frontend Docker image...'

                sh '''
                    docker build \
                      -t ${FRONTEND_REPO}:${IMAGE_TAG} \
                      ./frontend
                '''
            }
        }

        stage('Build Hello Service Image') {
            steps {
                echo 'Building Hello Service Docker image...'

                sh '''
                    docker build \
                      -t ${HELLO_REPO}:${IMAGE_TAG} \
                      ./backend/helloService
                '''
            }
        }

        stage('Build Profile Service Image') {
            steps {
                echo 'Building Profile Service Docker image...'

                sh '''
                    docker build \
                      -t ${PROFILE_REPO}:${IMAGE_TAG} \
                      ./backend/profileService
                '''
            }
        }

        // ============================================================
        // CI - ECR LOGIN
        // ============================================================

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

        // ============================================================
        // CI - TAG FOR ECR
        // ============================================================

        stage('Tag Images for ECR') {
            steps {

                echo 'Tagging Docker images for Amazon ECR...'

                sh '''
                    docker tag \
                      ${FRONTEND_REPO}:${IMAGE_TAG} \
                      ${FRONTEND_IMAGE}:${IMAGE_TAG}

                    docker tag \
                      ${HELLO_REPO}:${IMAGE_TAG} \
                      ${HELLO_IMAGE}:${IMAGE_TAG}

                    docker tag \
                      ${PROFILE_REPO}:${IMAGE_TAG} \
                      ${PROFILE_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        // ============================================================
        // CI - PUSH TO ECR
        // ============================================================

        stage('Push Images to ECR') {
            steps {

                echo 'Pushing Docker images to Amazon ECR...'

                sh '''
                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}

                    docker push ${HELLO_IMAGE}:${IMAGE_TAG}

                    docker push ${PROFILE_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        // ============================================================
        // CD - COMPLETE EKS DEPLOYMENT
        // ============================================================

        stage('CD - Deploy to EKS') {
            steps {

                /*
                 * One credential scope for the complete CD process.
                 *
                 * AWS credentials are required because:
                 *
                 * 1. aws eks update-kubeconfig
                 * 2. kubectl uses AWS authentication to communicate
                 *    with the EKS API server.
                 *
                 * Keeping this block around the entire CD process
                 * avoids repeating withCredentials for every stage.
                 */

                withCredentials([
                    usernamePassword(
                        credentialsId: env.AWS_CREDENTIALS_ID,
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "================================================"
                        echo "CD - DEPLOY TO EKS"
                        echo "================================================"

                        # ------------------------------------------------
                        # 1. Configure kubectl for EKS
                        # ------------------------------------------------

                        echo ""
                        echo "Configuring kubectl for EKS..."

                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER_NAME}

                        echo ""
                        echo "Current Kubernetes context:"
                        kubectl config current-context

                        echo ""
                        echo "Checking Kubernetes deployment access:"
                        kubectl get deployments -n default

                        # ------------------------------------------------
                        # 2. Apply Kubernetes manifests
                        # ------------------------------------------------

                        echo ""
                        echo "================================================"
                        echo "APPLYING KUBERNETES MANIFESTS"
                        echo "================================================"

                        echo "Manifest path: ${K8S_MANIFEST_PATH}"

                        kubectl apply \
                          -f ${K8S_MANIFEST_PATH}

                        # ------------------------------------------------
                        # 3. Deploy exact image version
                        # ------------------------------------------------

                        echo ""
                        echo "================================================"
                        echo "DEPLOYING EXACT IMAGE VERSION"
                        echo "================================================"

                        echo "Image tag: ${IMAGE_TAG}"

                        echo ""
                        echo "Updating Frontend deployment..."

                        kubectl set image deployment/frontend \
                          frontend=${FRONTEND_IMAGE}:${IMAGE_TAG}

                        echo ""
                        echo "Updating Hello Service deployment..."

                        kubectl set image deployment/hello-service \
                          hello-service=${HELLO_IMAGE}:${IMAGE_TAG}

                        echo ""
                        echo "Updating Profile Service deployment..."

                        kubectl set image deployment/profile-service \
                          profile-service=${PROFILE_IMAGE}:${IMAGE_TAG}

                        # ------------------------------------------------
                        # 4. Wait for Kubernetes rollout
                        # ------------------------------------------------

                        echo ""
                        echo "================================================"
                        echo "WAITING FOR ROLLOUT"
                        echo "================================================"

                        echo ""
                        echo "Frontend rollout..."

                        kubectl rollout status \
                          deployment/frontend \
                          --timeout=300s

                        echo ""
                        echo "Hello Service rollout..."

                        kubectl rollout status \
                          deployment/hello-service \
                          --timeout=300s

                        echo ""
                        echo "Profile Service rollout..."

                        kubectl rollout status \
                          deployment/profile-service \
                          --timeout=300s

                        # ------------------------------------------------
                        # 5. Validate deployment
                        # ------------------------------------------------

                        echo ""
                        echo "================================================"
                        echo "KUBERNETES DEPLOYMENT VALIDATION"
                        echo "================================================"

                        echo ""
                        echo "Deployments:"

                        kubectl get deployments -n default

                        echo ""
                        echo "Pods:"

                        kubectl get pods -n default -o wide

                        echo ""
                        echo "Services:"

                        kubectl get services -n default

                        echo ""
                        echo "Ingress:"

                        kubectl get ingress -n default

                        # ------------------------------------------------
                        # 6. Verify exact deployed images
                        # ------------------------------------------------

                        echo ""
                        echo "================================================"
                        echo "DEPLOYED IMAGES"
                        echo "================================================"

                        echo ""
                        echo "Frontend:"
                        kubectl get deployment/frontend \
                          -n default \
                          -o jsonpath='{.spec.template.spec.containers[0].image}'
                        echo ""

                        echo ""
                        echo "Hello Service:"
                        kubectl get deployment/hello-service \
                          -n default \
                          -o jsonpath='{.spec.template.spec.containers[0].image}'
                        echo ""

                        echo ""
                        echo "Profile Service:"
                        kubectl get deployment/profile-service \
                          -n default \
                          -o jsonpath='{.spec.template.spec.containers[0].image}'
                        echo ""

                        # ------------------------------------------------
                        # 7. Verify expected image tag
                        # ------------------------------------------------

                        echo ""
                        echo "================================================"
                        echo "EXPECTED IMAGE TAG"
                        echo "================================================"

                        echo "${IMAGE_TAG}"

                        echo ""
                        echo "================================================"
                        echo "CD DEPLOYMENT COMPLETED SUCCESSFULLY"
                        echo "================================================"
                    '''
                }
            }
        }
    }

    // ================================================================
    // POST ACTIONS
    // ================================================================

    post {

        success {

            echo '''
            ============================================================
                         CI/CD PIPELINE SUCCESSFUL!
            ============================================================

            Source code checked out successfully.

            Docker images built successfully.

            Images pushed to Amazon ECR.

            Kubernetes manifests applied successfully.

            Exact Jenkins build image deployed to EKS.

            Kubernetes rollouts completed successfully.

            Deployment validation completed successfully.

            Application is ready for browser testing.

            ============================================================
            '''
        }

        failure {

            echo '''
            ============================================================
                           CI/CD PIPELINE FAILED
            ============================================================

            Check the Jenkins Console Output for the failed stage.

            ============================================================
            '''
        }

        always {

            sh '''
                echo "Cleaning unused Docker images..."

                docker image prune -f || true
            '''
        }
    }
}