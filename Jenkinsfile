pipeline {
  agent any
  environment {
    AWS_REGION      = 'ap-south-1'
    EKS_CLUSTER     = 'trend-eks'
    KUBE_NAMESPACE  = 'trend'
    DOCKER_IMAGE    = 'ravimccullum/trend'   // <- replace
    DOCKERHUB_CREDS = 'dockerhub-cred'           // Jenkins credential id
  }
  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build & Tag Docker') {
      steps {
        script {
          TAG = "build-${env.BUILD_NUMBER}"
          sh "docker build -t ${DOCKER_IMAGE}:${TAG} ."
          sh "docker tag ${DOCKER_IMAGE}:${TAG} ${DOCKER_IMAGE}:latest"
        }
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh 'echo "${DH_PASS}" | docker login -u "${DH_USER}" --password-stdin'
          sh "docker push ${DOCKER_IMAGE}:${TAG}"
          sh "docker push ${DOCKER_IMAGE}:latest"
        }
      }
    }

    stage('Configure kube') {
      steps {
        sh "aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}"
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh "kubectl apply -f k8s/namespace.yaml || true"
        sh "sed 's|IMAGE_PLACEHOLDER|${DOCKER_IMAGE}:${TAG}|' k8s/deployment.yaml > k8s/_deployment.yaml"
        sh "kubectl apply -f k8s/_deployment.yaml"
        sh "kubectl apply -f k8s/service.yaml"
        sh "kubectl -n ${KUBE_NAMESPACE} rollout status deployment/trend"
      }
    }
  }
}
