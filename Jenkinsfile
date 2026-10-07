pipeline {
  agent any

  triggers {
    githubPush()
  }

  environment {
    AWS_REGION = 'ap-southeast-2'
    AWS_ACCOUNT_ID = '075810104493'
    ECR_REPO = 'boutique-frontend'
    IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    IMAGE_TAG = "${BUILD_NUMBER}"
    EKS_CLUSTER = 'devops-project2'
    NAMESPACE = 'boutique'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Validate Source') {
      steps {
        sh '''
          set -e
          test -f src/frontend/Dockerfile
          test -f helm-chart/Chart.yaml
          helm lint ./helm-chart -f helm-aws-values.yaml
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker build -t ${IMAGE_URI}:${IMAGE_TAG} ./src/frontend'
      }
    }

    stage('Trivy Scan') {
      steps {
        sh '''
          trivy image --severity HIGH,CRITICAL --ignore-status fixed --exit-code 1 ${IMAGE_URI}:${IMAGE_TAG}
        '''
      }
    }

    stage('Push ECR') {
      steps {
        sh '''
          set -e
          aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
          docker push ${IMAGE_URI}:${IMAGE_TAG}
        '''
      }
    }

    stage('Deploy with Helm') {
      steps {
        sh '''
          set -e
          aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}
          helm upgrade --install boutique ./helm-chart \
            --namespace ${NAMESPACE} \
            --create-namespace \
            -f helm-aws-values.yaml \
            --set frontend.image.repository=${IMAGE_URI} \
            --set frontend.image.tag=${IMAGE_TAG}
        '''
      }
    }

    stage('Verify') {
      steps {
        sh '''
          kubectl rollout status deployment/frontend -n ${NAMESPACE} --timeout=300s
          kubectl get pods -n ${NAMESPACE}
          kubectl get svc -n ${NAMESPACE}
        '''
      }
    }
  }
}
