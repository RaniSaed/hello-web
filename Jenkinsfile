pipeline {
  agent any

  environment {
    DOCKERHUB_REPO = 'rani19/hello-web'       
    DOCKER_CRED_ID = 'dockerhub-creds'        
    GIT_CRED_ID    = 'github-creds'           
    CONFIG_REPO    = 'https://github.com/RaniSaed>/hello-web-config.git'
    CONFIG_PATH    = 'k8s/deployment.yaml'    
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Image') {
      steps {
        script {
          def shortSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.IMAGE_TAG = shortSha
          sh """
            docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} -t ${DOCKERHUB_REPO}:latest .
          """
        }
      }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: DOCKER_CRED_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh """
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
            docker push ${DOCKERHUB_REPO}:latest
          """
        }
      }
    }

    stage('Update Config Repo') {
      steps {
        dir('hello-web-config') {
          git url: CONFIG_REPO, branch: 'main', credentialsId: GIT_CRED_ID
          // Replace image tag in deployment manifest
          sh """
            sed -i.bak -E "s|(image:\s*${DOCKERHUB_REPO}:).*|\\1${IMAGE_TAG}|" ${CONFIG_PATH}
            rm -f ${CONFIG_PATH}.bak
            git config user.email "Rani.saed19@gmail.com"
            git config user.name  "Jenkins CI"
            git add ${CONFIG_PATH}
            git commit -m "ci: update image tag to ${IMAGE_TAG}"
            git push origin main
          """
        }
      }
    }
  }
}