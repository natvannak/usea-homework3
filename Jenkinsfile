pipeline {
    agent any
    environment {
        AWS_REGION      = 'us-east-1'
        ECR_REGISTRY    = '464604123652.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY  = 'usea-homework2'
        IMAGE_TAG       = "${BUILD_NUMBER}"
        ECR_IMAGE       = "${ECR_REGISTRY}/${ECR_REPOSITORY}"

        SWARM_MANAGER   = '184.73.20.152'
        SSH_CREDENTIALS = 'swarm-ssh-key'

        PATH = '/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // stage('Check Docker & AWS') {
        //     steps {
        //         sh '''
        //             set -e

        //             echo "======================================"
        //             echo "Checking Docker"
        //             echo "======================================"
        //             docker --version

        //             echo ""
        //             echo "======================================"
        //             echo "Checking Buildx"
        //             echo "======================================"
        //             docker buildx version

        //             echo ""
        //             echo "======================================"
        //             echo "Checking AWS CLI"
        //             echo "======================================"
        //             aws --version
        //         '''
        //     }
        // }

        stage('ECR Login') {
            steps {
                sh '''
                    set -e

                    aws ecr get-login-password \
                    --region ${AWS_REGION} | \
                    docker login \
                    --username AWS \
                    --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Build & Push Image') {
            steps {
                sh '''
                    set -e

                    docker buildx build \
                      --platform linux/amd64,linux/arm64 \
                      -t ${ECR_IMAGE}:${IMAGE_TAG} \
                      --push .
                '''
            }
        }

        stage('Verify Image') {
            steps {
                sh '''
                    docker buildx imagetools inspect \
                    ${ECR_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Prepare Stack') {
            steps {
                sh '''
                    set -e
                    sed "s|IMAGE_PLACEHOLDER|${ECR_IMAGE}:${IMAGE_TAG}|g" \
                    docker-stack.yml > docker-stack-deploy.yml
                    cat docker-stack-deploy.yml
                '''
            }
        }

        // stage('Test SSH') {
        //     steps {

        //         withCredentials([
        //             sshUserPrivateKey(
        //                 credentialsId: "${SSH_CREDENTIALS}",
        //                 keyFileVariable: 'SSH_KEY',
        //                 usernameVariable: 'SSH_USER'
        //             )
        //         ]) {

        //             sh '''
        //                 set -e

        //                 echo "======================================"
        //                 echo "Testing SSH"
        //                 echo "======================================"

        //                 ssh \
        //                   -i "$SSH_KEY" \
        //                   -o StrictHostKeyChecking=no \
        //                   -o UserKnownHostsFile=/dev/null \
        //                   "$SSH_USER@$SWARM_MANAGER" \
        //                   "hostname"
        //             '''
        //         }
        //     }
        // }

        stage('Copy Stack To Swarm') {
            steps {

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: "${SSH_CREDENTIALS}",
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                        set -e

                        scp \
                          -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=no \
                          -o UserKnownHostsFile=/dev/null \
                          docker-stack-deploy.yml \
                          "$SSH_USER@$SWARM_MANAGER:/tmp/docker-stack-deploy.yml"
                    '''
                }
            }
        }
        stage('Deploy Stack') {
            steps {

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: "${SSH_CREDENTIALS}",
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                        set -eux

                        ssh \
                        -i "$SSH_KEY" \
                        -o IdentitiesOnly=yes \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        "$SSH_USER@$SWARM_MANAGER" \
                        "
                        sudo docker stack deploy \
                            --with-registry-auth \
                            -c /tmp/docker-stack-deploy.yml \
                            homework2

                        sudo docker service ls
                        "
                    '''
                }
            }
        }

        stage('Verify Services') {
            steps {

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: "${SSH_CREDENTIALS}",
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                        set -e

                        ssh \
                          -i "$SSH_KEY" \
                          -o StrictHostKeyChecking=no \
                          -o UserKnownHostsFile=/dev/null \
                          "$SSH_USER@$SWARM_MANAGER" \
                          "
                          docker stack services homework2
                          "
                    '''
                }
            }
        }
    }

    post {

        success {
            echo "======================================"
            echo "CI/CD Pipeline Success"
            echo "======================================"
        }

        failure {
            echo "======================================"
            echo "CI/CD Pipeline Failed"
            echo "======================================"
        }

        always {
            sh 'rm -f docker-stack-deploy.yml || true'
        }
    }
}