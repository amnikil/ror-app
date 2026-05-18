pipeline {
  agent any

  environment {
    IMAGE_NAME = "ror-app"
    IMAGE_TAG  = "${BUILD_NUMBER}"
  }

  stages {

    stage('1. Checkout') {
      steps {
        checkout scm
        echo "✅ Code checked out"
      }
    }

    stage('2. SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          script {
            def scannerHome = tool 'SonarScanner'
            sh "${scannerHome}/bin/sonar-scanner"
          }
        }
        echo "✅ SonarQube scan done"
      }
    }

    stage('3. Docker Build') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
        sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
        echo "✅ Docker image built"
      }
    }

    stage('4. Trivy Scan') {
      steps {
        sh "trivy image --exit-code 0 --severity HIGH,CRITICAL ${IMAGE_NAME}:latest || true"
        echo "✅ Security scan done"
      }
    }

    stage('5. Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
          sh "docker tag ${IMAGE_NAME}:latest ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
          sh "docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
          sh "docker push ${DOCKER_USER}/${IMAGE_NAME}:latest"
          echo "✅ Image pushed to DockerHub"
        }
      }
    }

    stage('6. Update Helm Tag') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github-creds',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_PASS'
        )]) {
          sh """
            sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' helm/ror-app/values.yaml
            git config user.email 'jenkins@ci.com'
            git config user.name 'Jenkins'
            git add helm/ror-app/values.yaml
            git commit -m 'CI: Update image tag to ${IMAGE_TAG}'
            git push https://${GIT_USER}:${GIT_PASS}@github.com/amnikil/ror-app.git main
          """
          echo "✅ Helm values updated — ArgoCD will auto-deploy"
        }
      }
    }

  }

  post {
    success { echo '🎉 PIPELINE SUCCESS!' }
    failure { echo '❌ PIPELINE FAILED - Check logs' }
  }
}
