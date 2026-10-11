pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        buildDiscarder(
            logRotator(
                numToKeepStr: '20',
                artifactNumToKeepStr: '10'
            )
        )
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: [
                'build',
                'push',
                'deploy',
                'rollback'
            ]
        )

        gitParameter(
            name: 'VERSION',
            type: 'PT_TAG',
            defaultValue: '',
            description: 'Select Git tag, for example v1.0.1',
            tagFilter: '*',
            sortMode: 'DESCENDING_SMART',
            selectedValue: 'TOP',
            quickFilterEnabled: true
        )
    }

    environment {
        AWS_REGION      = 'us-east-1'
        AWS_ACCOUNT_ID  = '464604123652'
        ECR_REPO        = 'usea-homework3'

        ECR_REGISTRY    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE       = "${ECR_REGISTRY}/${ECR_REPO}"

        MANAGER_HOST    = '54.82.194.125'
        MANAGER_USER    = 'ubuntu'

        SERVICE_NAME    = 'homework3_web'

        AWS_CREDENTIALS = 'aws-credentials'
        SSH_CREDENTIALS = 'swarm-ssh-key'

        PATH = '/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin'
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    def action  = params.ACTION?.trim()
                    def version = params.VERSION?.trim()

                    echo '======================================'
                    echo "ACTION  : ${action}"
                    echo "VERSION : ${version ?: '(not selected)'}"
                    echo '======================================'

                    if (!action) {
                        error('ACTION is required.')
                    }

                    if (action in ['build', 'push', 'deploy', 'rollback']) {
                        if (!version) {
                            error(
                                'VERSION is required. ' +
                                'Create and push a Git tag, then select it.'
                            )
                        }

                        if (!(version ==~ /^[A-Za-z0-9][A-Za-z0-9._-]*$/)) {
                            error(
                                "Invalid VERSION '${version}'. " +
                                'Use letters, numbers, dots, underscores or hyphens.'
                            )
                        }
                    }

                    if (action == 'deploy' || action == 'rollback') {
                        if (!env.MANAGER_HOST?.trim()) {
                            error('MANAGER_HOST is not configured.')
                        }

                        if (env.MANAGER_HOST == 'YOUR_SWARM_MANAGER_PUBLIC_IP') {
                            error(
                                'Replace YOUR_SWARM_MANAGER_PUBLIC_IP ' +
                                'with the real Docker Swarm Manager IP.'
                            )
                        }

                        if (!env.SERVICE_NAME?.trim()) {
                            error('SERVICE_NAME is not configured.')
                        }
                    }

                    /*
                     * Make VERSION available to all following stages.
                     */
                    env.IMAGE_TAG = version ?: ''
                    env.FULL_IMAGE_NAME = version
                        ? "${env.ECR_IMAGE}:${version}"
                        : ''

                    echo "IMAGE TAG : ${env.IMAGE_TAG ?: '(not required)'}"
                    echo "IMAGE     : ${env.FULL_IMAGE_NAME ?: '(not required)'}"
                }
            }
        }

        stage('Checkout Selected Version') {
            when {
                expression {
                    params.ACTION in ['build', 'push']
                }
            }

            steps {
                checkout scm

                sh '''
                    set -eu

                    echo "Fetching Git tags..."
                    git fetch --tags --force origin

                    echo "Checking selected tag: ${IMAGE_TAG}"

                    if ! git rev-parse -q --verify \
                        "refs/tags/${IMAGE_TAG}^{commit}" >/dev/null
                    then
                        echo "ERROR: Git tag '${IMAGE_TAG}' does not exist."
                        exit 1
                    fi

                    echo "Checking out Git tag: ${IMAGE_TAG}"
                    git checkout --force "refs/tags/${IMAGE_TAG}"

                    echo "======================================"
                    echo "Selected Git information"
                    echo "======================================"
                    echo "Tag       : ${IMAGE_TAG}"
                    echo "Commit    : $(git rev-parse --short HEAD)"
                    echo "Description: $(git describe --tags --always)"
                    echo "======================================"
                '''
            }
        }

        stage('Check Required Commands') {
            when {
                expression {
                    params.ACTION in ['build', 'push']
                }
            }

            steps {
                sh '''
                    set -eu

                    echo "Checking required commands..."

                    command -v git >/dev/null 2>&1 || {
                        echo "ERROR: git is not installed."
                        exit 1
                    }

                    command -v docker >/dev/null 2>&1 || {
                        echo "ERROR: docker is not installed."
                        exit 1
                    }

                    docker version

                    if [ "${ACTION}" = "push" ] ||
                       [ "${ACTION}" = "deploy" ]
                    then
                        command -v aws >/dev/null 2>&1 || {
                            echo "ERROR: AWS CLI is not installed."
                            exit 1
                        }

                        aws --version
                    fi
                '''
            }
        }

        stage('Build Docker Image') {
            when {
                expression {
                    params.ACTION == 'build'
                }
            }

            steps {
                sh '''
                    set -eu

                    echo "======================================"
                    echo "Building Docker image"
                    echo "Image: ${FULL_IMAGE_NAME}"
                    echo "======================================"

                    docker buildx build \
                        --platform linux/amd64 \
                        --pull \
                        --tag "${FULL_IMAGE_NAME}" \
                        --load \
                        .

                    docker image inspect \
                        "${FULL_IMAGE_NAME}" \
                        --format \
                        'Image ID: {{.Id}} | Created: {{.Created}}'
                '''
            }
        }

        stage('Push to AWS ECR') {
            when {
                expression {
                    params.ACTION == 'push'
                }
            }

            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS}"
                    ]
                ]) {
                    sh '''
                        set -eu

                        echo "======================================"
                        echo "Checking AWS credentials"
                        echo "======================================"

                        aws sts get-caller-identity

                        echo "======================================"
                        echo "Checking ECR repository"
                        echo "Repository: ${ECR_REPO}"
                        echo "Region    : ${AWS_REGION}"
                        echo "======================================"

                        aws ecr describe-repositories \
                            --region "${AWS_REGION}" \
                            --repository-names "${ECR_REPO}" \
                            >/dev/null

                        echo "======================================"
                        echo "Logging in to AWS ECR"
                        echo "Registry: ${ECR_REGISTRY}"
                        echo "======================================"

                        aws ecr get-login-password \
                            --region "${AWS_REGION}" \
                        | docker login \
                            --username AWS \
                            --password-stdin "${ECR_REGISTRY}"

                        echo "======================================"
                        echo "Pushing Docker image"
                        echo "Image: ${FULL_IMAGE_NAME}"
                        echo "======================================"

                        docker push "${FULL_IMAGE_NAME}"

                        echo "======================================"
                        echo "Verifying image in AWS ECR"
                        echo "======================================"

                        aws ecr describe-images \
                            --region "${AWS_REGION}" \
                            --repository-name "${ECR_REPO}" \
                            --image-ids imageTag="${IMAGE_TAG}" \
                            --query \
                            'imageDetails[0].[imageTags,imageDigest,imagePushedAt]' \
                            --output table

                        echo "Successfully pushed:"
                        echo "${FULL_IMAGE_NAME}"
                    '''
                }
            }
        }

        stage('Test SSH Connection') {
            when {
                anyOf {
                    expression {
                        params.ACTION == 'deploy'
                    }

                    expression {
                        params.ACTION == 'rollback'
                    }
                }
            }

            steps {
                sshagent(["${SSH_CREDENTIALS}"]) {
                    sh '''
                        set -eu

                        echo "======================================"
                        echo "Testing SSH connection"
                        echo "Manager: ${MANAGER_USER}@${MANAGER_HOST}"
                        echo "======================================"

                        ssh \
                            -o BatchMode=yes \
                            -o ConnectTimeout=10 \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "${MANAGER_USER}@${MANAGER_HOST}" \
                            '
                                set -eu

                                echo "Hostname: $(hostname)"
                                echo "User    : $(whoami)"

                                command -v docker >/dev/null 2>&1 || {
                                    echo "ERROR: Docker is not installed."
                                    exit 1
                                }

                                docker info >/dev/null

                                echo "Docker connection successful."
                            '
                    '''
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            when {
                expression {
                    params.ACTION == 'deploy'
                }
            }

            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "${AWS_CREDENTIALS}"
                    ]
                ]) {
                    sshagent(["${SSH_CREDENTIALS}"]) {
                        sh '''
                            set -eu

                            TOKEN_FILE="$WORKSPACE/ecr-token.txt"

                            cleanup() {
                                rm -f "$TOKEN_FILE"
                            }

                            trap cleanup EXIT INT TERM

                            echo "Creating temporary ECR login token..."

                            aws ecr get-login-password \
                                --region "${AWS_REGION}" \
                                > "$TOKEN_FILE"

                            chmod 600 "$TOKEN_FILE"

                            echo "Copying ECR token to Swarm Manager..."

                            scp \
                                -o BatchMode=yes \
                                -o ConnectTimeout=10 \
                                -o StrictHostKeyChecking=no \
                                -o UserKnownHostsFile=/dev/null \
                                "$TOKEN_FILE" \
                                "${MANAGER_USER}@${MANAGER_HOST}:/tmp/homework3-ecr-token.txt"

                            echo "Deploying ${FULL_IMAGE_NAME}..."

                            ssh \
                                -o BatchMode=yes \
                                -o ConnectTimeout=10 \
                                -o StrictHostKeyChecking=no \
                                -o UserKnownHostsFile=/dev/null \
                                "${MANAGER_USER}@${MANAGER_HOST}" \
                                "
                                    set -eu

                                    TOKEN_FILE='/tmp/homework3-ecr-token.txt'

                                    cleanup_remote() {
                                        rm -f \\"\\$TOKEN_FILE\\"
                                    }

                                    trap cleanup_remote EXIT INT TERM

                                    echo 'Logging Docker into AWS ECR...'

                                    cat \\"\\$TOKEN_FILE\\" |
                                    docker login \\
                                        --username AWS \\
                                        --password-stdin \\
                                        '${ECR_REGISTRY}'

                                    echo 'Checking Docker Swarm status...'

                                    SWARM_STATE=\\$(docker info \\
                                        --format '{{.Swarm.LocalNodeState}}')

                                    if [ \\"\\$SWARM_STATE\\" != 'active' ]; then
                                        echo 'ERROR: Docker Swarm is not active.'
                                        exit 1
                                    fi

                                    echo 'Checking service: ${SERVICE_NAME}'

                                    if ! docker service inspect \\
                                        '${SERVICE_NAME}' >/dev/null 2>&1
                                    then
                                        echo 'ERROR: Docker service not found:'
                                        echo '${SERVICE_NAME}'
                                        echo
                                        echo 'Available services:'
                                        docker service ls
                                        exit 1
                                    fi

                                    echo 'Updating Docker Swarm service...'

                                    docker service update \\
                                        --with-registry-auth \\
                                        --image '${FULL_IMAGE_NAME}' \\
                                        --update-parallelism 1 \\
                                        --update-delay 10s \\
                                        --update-failure-action rollback \\
                                        --detach=false \\
                                        '${SERVICE_NAME}'

                                    echo 'Deployment completed successfully.'
                                "
                        '''
                    }
                }
            }
        }


        stage('Rollback Docker Swarm') {
            when {
                expression {
                    params.ACTION == 'rollback'
                }
            }

            steps {
                script {
                    def rollbackVersion = params.VERSION?.trim()

                    if (!rollbackVersion) {
                        error(
                            'VERSION is required for rollback. ' +
                            'Select the image version to restore.'
                        )
                    }

                    if (!(rollbackVersion ==~ /^[A-Za-z0-9][A-Za-z0-9._-]*$/)) {
                        error(
                            "Invalid rollback VERSION '${rollbackVersion}'."
                        )
                    }

                    env.IMAGE_TAG = rollbackVersion
                    env.FULL_IMAGE_NAME =
                        "${env.ECR_IMAGE}:${rollbackVersion}"

                    echo "Rollback version: ${env.IMAGE_TAG}"
                    echo "Rollback image  : ${env.FULL_IMAGE_NAME}"
                }

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {
                    sshagent(['swarm-ssh-key']) {
                        sh '''
                            set -eu

                            TOKEN_FILE="${WORKSPACE}/ecr-token.txt"
                            REMOTE_TOKEN="/tmp/homework3-rollback-token.txt"

                            cleanup_local() {
                                rm -f "${TOKEN_FILE}"
                            }

                            trap cleanup_local EXIT INT TERM

                           
                            aws ecr describe-images \
                                --region "${AWS_REGION}" \
                                --repository-name "${ECR_REPO}" \
                                --image-ids imageTag="${IMAGE_TAG}" \
                                >/dev/null

                            echo "Creating temporary ECR token..."

                            aws ecr get-login-password \
                                --region "${AWS_REGION}" \
                                > "${TOKEN_FILE}"

                            chmod 600 "${TOKEN_FILE}"

                            echo "Copying ECR token to Swarm Manager..."

                            scp \
                                -o BatchMode=yes \
                                -o ConnectTimeout=15 \
                                -o StrictHostKeyChecking=no \
                                -o UserKnownHostsFile=/dev/null \
                                "${TOKEN_FILE}" \
                                "${MANAGER_USER}@${MANAGER_HOST}:${REMOTE_TOKEN}"

                            echo "Starting selected-version rollback..."

                            ssh \
                                -o BatchMode=yes \
                                -o ConnectTimeout=15 \
                                -o StrictHostKeyChecking=no \
                                -o UserKnownHostsFile=/dev/null \
                                "${MANAGER_USER}@${MANAGER_HOST}" \
                                bash -s -- \
                                "${SERVICE_NAME}" \
                                "${FULL_IMAGE_NAME}" \
                                "${ECR_REGISTRY}" \
                                "${REMOTE_TOKEN}" <<'REMOTE_SCRIPT'
                                set -eu

                                SERVICE_NAME="$1"
                                FULL_IMAGE_NAME="$2"
                                ECR_REGISTRY="$3"
                                REMOTE_TOKEN="$4"

                                cleanup_remote() {
                                    rm -f "${REMOTE_TOKEN}"
                                }

                                trap cleanup_remote EXIT INT TERM

                                echo "======================================"
                                echo "Selected-Version Rollback"
                                echo "======================================"
                                echo "Service : ${SERVICE_NAME}"
                                echo "Image   : ${FULL_IMAGE_NAME}"

                                if [ ! -f "${REMOTE_TOKEN}" ]; then
                                    echo "ERROR: ECR token file was not received."
                                    exit 1
                                fi

                                echo "Logging Docker into AWS ECR..."

                                docker login \
                                    --username AWS \
                                    --password-stdin \
                                    "${ECR_REGISTRY}" \
                                    < "${REMOTE_TOKEN}"

                                echo "Checking Docker Swarm state..."

                                SWARM_STATE="$(docker info \
                                    --format '{{.Swarm.LocalNodeState}}')"

                                CONTROL_AVAILABLE="$(docker info \
                                    --format '{{.Swarm.ControlAvailable}}')"

                                echo "Swarm state      : ${SWARM_STATE}"
                                echo "Control available: ${CONTROL_AVAILABLE}"

                                if [ "${SWARM_STATE}" != "active" ]; then
                                    echo "ERROR: Docker Swarm is not active."
                                    exit 1
                                fi

                                if [ "${CONTROL_AVAILABLE}" != "true" ]; then
                                    echo "ERROR: This server is not a Swarm Manager."
                                    exit 1
                                fi

                                echo "Checking service: ${SERVICE_NAME}"

                                if ! docker service inspect \
                                    "${SERVICE_NAME}" \
                                    >/dev/null 2>&1
                                then
                                    echo "ERROR: Service does not exist: ${SERVICE_NAME}"
                                    echo
                                    echo "Available services:"
                                    docker service ls
                                    exit 1
                                fi

                                CURRENT_IMAGE_BEFORE="$(docker service inspect \
                                    --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}' \
                                    "${SERVICE_NAME}")"

                                echo "Current image before update:"
                                echo "${CURRENT_IMAGE_BEFORE}"

                                echo "======================================"
                                echo "Restoring selected image"
                                echo "======================================"

                                docker service update \
                                    --with-registry-auth \
                                    --image "${FULL_IMAGE_NAME}" \
                                    --replicas 2 \
                                    --update-parallelism 1 \
                                    --update-delay 10s \
                                    --update-order start-first \
                                    --update-failure-action rollback \
                                    --rollback-parallelism 1 \
                                    --rollback-order stop-first \
                                    --restart-condition any \
                                    --restart-delay 5s \
                                    --detach=false \
                                    "${SERVICE_NAME}"

                                CURRENT_IMAGE_AFTER="$(docker service inspect \
                                    --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}' \
                                    "${SERVICE_NAME}")"

                                echo "======================================"
                                echo "Verifying selected image"
                                echo "======================================"
                                echo "Expected: ${FULL_IMAGE_NAME}"
                                echo "Current : ${CURRENT_IMAGE_AFTER}"

                                case "${CURRENT_IMAGE_AFTER}" in
                                    "${FULL_IMAGE_NAME}"|"${FULL_IMAGE_NAME}@sha256:"*)
                                        echo "Selected version restored successfully."
                                        ;;
                                    *)
                                        echo "ERROR: Service is using an unexpected image."
                                        docker service ps --no-trunc "${SERVICE_NAME}"
                                        exit 1
                                        ;;
                                esac

                                EXPECTED_REPLICAS="$(docker service inspect \
                                    --format '{{.Spec.Mode.Replicated.Replicas}}' \
                                    "${SERVICE_NAME}")"

                                ATTEMPT=1
                                MAX_ATTEMPTS=30

                                while [ "${ATTEMPT}" -le "${MAX_ATTEMPTS}" ]
                                do
                                    RUNNING_REPLICAS="$(docker service ps \
                                        --filter desired-state=running \
                                        --format '{{.CurrentState}}' \
                                        "${SERVICE_NAME}" |
                                        grep -c '^Running' || true)"

                                    echo "Replica check ${ATTEMPT}/${MAX_ATTEMPTS}: ${RUNNING_REPLICAS}/${EXPECTED_REPLICAS}"

                                    if [ "${RUNNING_REPLICAS}" -eq "${EXPECTED_REPLICAS}" ]; then
                                        echo "All replicas are running."
                                        break
                                    fi

                                    if [ "${ATTEMPT}" -eq "${MAX_ATTEMPTS}" ]; then
                                        echo "ERROR: Service replicas did not become healthy."
                                        docker service ps --no-trunc "${SERVICE_NAME}"
                                        exit 1
                                    fi

                                    ATTEMPT=$((ATTEMPT + 1))
                                    sleep 5
                                done

                                echo "======================================"
                                echo "Running service tasks"
                                echo "======================================"

                                docker service ps \
                                    --filter desired-state=running \
                                    "${SERVICE_NAME}"

                                echo "======================================"
                                echo "Selected-version rollback completed"
                                echo "======================================"
REMOTE_SCRIPT
                        '''
                    }
                }
            }
        }

        stage('Verify Deployment') {
            when {
                anyOf {
                    expression {
                        params.ACTION == 'deploy'
                    }

                    expression {
                        params.ACTION == 'rollback'
                    }
                }
            }

            steps {
                sshagent(["${SSH_CREDENTIALS}"]) {
                    sh '''
                        set -eu

                        ssh \
                            -o BatchMode=yes \
                            -o ConnectTimeout=10 \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "${MANAGER_USER}@${MANAGER_HOST}" \
                            "
                                set -eu

                                echo '======================================'
                                echo 'Docker Swarm Service'
                                echo '======================================'

                                docker service ls \
                                    --filter name='${SERVICE_NAME}'

                                echo '======================================'
                                echo 'Current Docker Image'
                                echo '======================================'

                                docker service inspect \
                                    --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}' \
                                    '${SERVICE_NAME}'

                                echo '======================================'
                                echo 'Service Update Status'
                                echo '======================================'

                                docker service inspect \
                                    --format '{{json .UpdateStatus}}' \
                                    '${SERVICE_NAME}'

                                echo '======================================'
                                echo 'Service Tasks'
                                echo '======================================'

                                docker service ps \
                                    --no-trunc \
                                    '${SERVICE_NAME}'
                            "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo 'Pipeline completed successfully.'
            echo "ACTION : ${params.ACTION}"
            echo "VERSION: ${params.VERSION ?: '(not required)'}"
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'Pipeline failed.'
            echo 'Check the failed stage logs above.'
            echo '======================================'
        }

        aborted {
            echo 'Pipeline was aborted.'
        }

        always {
            sh '''
                rm -f \
                    "$WORKSPACE/ecr-token.txt" \
                    "$WORKSPACE/aws-session-token.txt" \
                    2>/dev/null || true
            '''

            cleanWs(
                cleanWhenNotBuilt: false,
                deleteDirs: true,
                disableDeferredWipeout: false,
                notFailBuild: true
            )
        }
    }
}