pipeline {
    agent any

    environment {
        AWS_REGION     = 'ap-south-1'
        ECR_REPOSITORY = 'seclock'
    }

    stages {

        stage('Checkout') {
            steps {
                git credentialsId: 'github-https',
                    url: 'https://github.com/lighacu/seclock.git',
                    branch: 'main'
            }
        }

        stage('Check GitOps Commit') {
            steps {
                script {
                    def commitMessage = sh(
                        script: "git log -1 --pretty=%s",
                        returnStdout: true
                    ).trim()

                    echo "Latest commit: ${commitMessage}"

                    if (commitMessage.startsWith("Deploy seclock ")) {
                        echo "This is a Jenkins GitOps deployment commit."
                        echo "Skipping this build to prevent a CI loop."

                        currentBuild.result = 'ABORTED'
                        error('Skipping Jenkins GitOps commit')
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    set -e

                    python3 -m venv venv
                    . venv/bin/activate

                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    . venv/bin/activate
                    python test_e2e.py
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([
                        string(
                            credentialsId: 'sonarqube',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -e

                            sonar-scanner \
                                -Dsonar.projectKey=seclock \
                                -Dsonar.projectName=Seclock \
                                -Dsonar.sources=. \
                                -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e

                    echo "Building Docker image..."

                    docker build \
                        -t ${ECR_REPOSITORY}:${BUILD_NUMBER} .

                    echo "Docker image built:"
                    docker images ${ECR_REPOSITORY}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                withAWS(
                    credentials: 'aws-cred-new',
                    region: "${AWS_REGION}"
                ) {
                    sh '''
                        set -e

                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                            --query Account \
                            --output text)

                        ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        echo "Logging into Amazon ECR..."
                        echo "Registry: ${ECR_REGISTRY}"

                        aws ecr get-login-password \
                            --region ${AWS_REGION} | \
                        docker login \
                            --username AWS \
                            --password-stdin \
                            ${ECR_REGISTRY}
                    '''
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                withAWS(
                    credentials: 'aws-cred-new',
                    region: "${AWS_REGION}"
                ) {
                    sh '''
                        set -e

                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                            --query Account \
                            --output text)

                        ECR_IMAGE=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${BUILD_NUMBER}

                        echo "Tagging image:"
                        echo "${ECR_IMAGE}"

                        docker tag \
                            ${ECR_REPOSITORY}:${BUILD_NUMBER} \
                            ${ECR_IMAGE}

                        echo "Pushing image to ECR..."

                        docker push ${ECR_IMAGE}

                        echo "Successfully pushed:"
                        echo "${ECR_IMAGE}"
                    '''
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                withAWS(
                    credentials: 'aws-cred-new',
                    region: "${AWS_REGION}"
                ) {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'github-https',
                            usernameVariable: 'GIT_USERNAME',
                            passwordVariable: 'GIT_PASSWORD'
                        )
                    ]) {
                        sh '''
                            set -e

                            AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                                --query Account \
                                --output text)

                            ECR_IMAGE=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${BUILD_NUMBER}

                            echo "======================================"
                            echo "Updating Kubernetes Manifest"
                            echo "======================================"
                            echo "New image:"
                            echo "${ECR_IMAGE}"

                            sed -i "s|image: .*|image: ${ECR_IMAGE}|" k8s/deployment.yaml

                            echo ""
                            echo "Updated deployment.yaml:"
                            grep "image:" k8s/deployment.yaml

                            git config user.name "Jenkins"
                            git config user.email "jenkins@localhost"

                            git add k8s/deployment.yaml

                            git commit \
                                -m "Deploy seclock ${BUILD_NUMBER}" \
                                || echo "No manifest changes detected"

                            echo ""
                            echo "Pushing manifest change to GitHub..."

                            git config credential.helper \
                                "!f() { echo username=\\$GIT_USERNAME; echo password=\\$GIT_PASSWORD; }; f"

                            git push origin HEAD:main

                            git config --unset credential.helper || true

                            echo ""
                            echo "GitOps manifest successfully pushed."
                        '''
                    }
                }
            }
        }

        stage('Verify GitOps Configuration') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "GitOps Configuration"
                    echo "======================================"

                    echo ""
                    echo "Deployment image:"
                    grep "image:" k8s/deployment.yaml

                    echo ""
                    echo "Latest Git commit:"
                    git log -1 --oneline

                    echo ""
                    echo "GitOps update completed successfully."
                    echo "Argo CD will now detect the Git change."
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo 'Seclock CI/CD Pipeline SUCCESS'
            echo '======================================'
            echo 'Docker image pushed to ECR.'
            echo 'Kubernetes manifest updated.'
            echo 'Argo CD will deploy the new image.'
        }

        failure {
            echo '======================================'
            echo 'Seclock CI/CD Pipeline FAILED'
            echo '======================================'
        }

        aborted {
            echo 'Jenkins build skipped/aborted.'
        }
    }
}
