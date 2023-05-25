pipeline {
  agent any
  environment {
    DOCKER_REGSITRY_USERNAME        = credentials('DOCKER_REGSITRY_USERNAME')
    DOCKER_REGISTRY_USERNAME    = credentials('DOCKER_REGISTRY_USERNAME')
  }

  stages {
    stage('Build') {
      steps {
        sh '''
          echo "Starting to build docker image"
          docker build -t orezfu/obo:v1.${BUILD_NUMBER} -f Dockerfile .
        '''
        sh '''
          echo "Starting to push docker image"
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGSITRY_USERNAME} --password-stdin
          docker push "orezfu/obo:v1.${BUILD_NUMBER}"
        '''
      }
    }
  }
}