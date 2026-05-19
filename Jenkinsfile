pipeline {
  agent any

  environment {
    IMAGE_NAME     = "ror-app"
    IMAGE_TAG      = "${BUILD_NUMBER}"
    DOCKERHUB_USER = "amnikil"
    SONAR_URL      = "http://172.20.0.2:9000"
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
            sh """
              ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=ror-app \
                -Dsonar.sources=app \
                -Dsonar.host.url=http://172.20.0.2:9000 \
                -Dsonar.exclusions=vendor/*,public/*
            """
          }
        }
        echo "✅ SonarQube scan complete"
      }
    }

    stage('3. Docker Build') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
        sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_USER}/${IMAGE_NAME}:latest"
        echo "✅ Docker image built"
      }
    }

    stage('4. Trivy Security Scan') {
      steps {
        sh "trivy image --exit-code 0 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG} || true"
        echo "✅ Security scan complete"
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
          sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
          sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:latest"
          echo "✅ Pushed to DockerHub"
        }
      }
    }

    stage('Update Helm Tag') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {

            sh '''
            git config user.email "jenkins@ci.com"
            git config user.name "Jenkins"

            git stash || true

            git checkout main
            git pull https://${GIT_USER}:${GIT_PASS}@github.com/amnikil/ror-app.git main

            sed -i "s/tag:.*/tag: \\"${BUILD_NUMBER}\\"/" helm/ror-app/values.yaml

            git add helm/ror-app/values.yaml

            git commit -m "Update image tag to ${BUILD_NUMBER}" || true

            git push https://${GIT_USER}:${GIT_PASS}@github.com/amnikil/ror-app.git main
            '''
        }
    }
}

  post {
    success { echo '🎉 FULL PIPELINE SUCCESS!' }
    failure { echo '❌ Pipeline failed - check logs' }
  }
}
