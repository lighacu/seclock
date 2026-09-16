pipeline {
agent any

environment {
    AWS_REGION = 'ap-south-1'
    ECR_REPOSITORY = 'seclock'
    EKS_CLUSTER = 'seclock-cluster'
}

stages {

  stage('Checkout') {
    steps {
        git credentialsId: 'github-https',
            url: 'https://github.com/lighacu/seclock.git',
            branch: 'main'
    }
}

    stage('Install Dependencies') {
        steps {
            sh '''
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
                . venv/bin/activate
                python test_e2e.py
            '''
        }
    }

    stage('SonarQube Analysis') {
        steps {
            withSonarQubeEnv('SonarQube') {
                sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=seclock \
                      -Dsonar.projectName=Seclock \
                      -Dsonar.sources=.
                '''
            }
        }
    }

    stage('Build Docker Image') {
        steps {
            sh '''
                docker build -t ${ECR_REPOSITORY}:${BUILD_NUMBER} .
            '''
        }
    }

    stage('Login to Amazon ECR') {
        steps {
            sh '''
                AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                    --query Account \
                    --output text)

                aws ecr get-login-password \
                    --region ${AWS_REGION} | \
                docker login \
                    --username AWS \
                    --password-stdin \
                    ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
            '''
        }
    }

    stage('Push Docker Image to ECR') {
        steps {
            sh '''
                AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                    --query Account \
                    --output text)

                ECR_IMAGE=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${BUILD_NUMBER}

                docker tag \
                    ${ECR_REPOSITORY}:${BUILD_NUMBER} \
                    ${ECR_IMAGE}

                docker push ${ECR_IMAGE}
            '''
        }
    }

    stage('Deploy to EKS') {
        steps {
            sh '''
                aws eks update-kubeconfig \
                    --region ${AWS_REGION} \
                    --name ${EKS_CLUSTER}

                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml
            '''
        }
    }

    stage('Verify Deployment') {
        steps {
            sh '''
                kubectl get pods -A
                kubectl get services -A
                kubectl get deployments -A
            '''
        }
    }
}

post {
    success {
        echo 'Seclock CI/CD pipeline completed successfully!'
    }

    failure {
        echo 'Seclock CI/CD pipeline failed.'
    }
}

}
