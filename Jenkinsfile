pipeline {
  agent any
  environment {
    IMAGE_NAME = "ror-app"
    DOCKERHUB_USER = "amnikil"
  }
  stages {
    stage('1. Checkout') {
      steps {
        checkout scm
        echo "Checkout done"
      }
    }
    stage('2. SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          script {
            def scannerHome = tool 'SonarScanner'
            sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=ror-app -Dsonar.sources=app -Dsonar.host.url=http://172.20.0.2:9000 -Dsonar.exclusions=vendor/*,public/*"
          }
        }
        echo "SonarQube done"
      }
    }
    stage('3. Docker Build') {
      steps {
        sh "docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER} ."
        sh "docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE_NAME}:latest"
        echo "Docker build done"
      }
    }
    stage('4. Trivy Scan') {
      steps {
        sh "trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE_NAME}:latest || true"
        echo "Trivy done"
      }
    }
    stage('5. Push DockerHub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
          sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}"
          sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:latest"
        }
        echo "DockerHub push done"
      }
    }
    stage('6. Update Helm Tag') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
          sh """
            git config user.email "jenkins@ci.com"
            git config user.name "Jenkins"
            git stash || true
            git checkout main
            git config pull.rebase false && git pull https://${GIT_USER}:${GIT_PASS}@github.com/amnikil/ror-app.git main
            sed -i 's/tag:.*/tag: "${BUILD_NUMBER}"/' helm/ror-app/values.yaml
            git add helm/ror-app/values.yaml
            git commit -m "CI: image tag ${BUILD_NUMBER}" || true
            git push https://${GIT_USER}:${GIT_PASS}@github.com/amnikil/ror-app.git main
          """
        }
        echo "Helm updated"
      }
    }
  }
  post {
    success { echo 'PIPELINE SUCCESS!' }
    failure { echo 'PIPELINE FAILED' }
  }
}
