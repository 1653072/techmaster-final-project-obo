pipeline {
  agent any
  environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
    FINAL_PROJECT_MANIFEST_REPO_URL = credentials('FINAL_PROJECT_MANIFEST_REPO_URL')
    FINAL_PROJECT_MANIFEST_REPO_NAME = credentials('FINAL_PROJECT_MANIFEST_REPO_NAME')
    APP_SERVICE_NODE_PORT_DEV = 30000
    APP_SERVICE_NODE_PORT_PRD = 30001
  }

  stages {
    // In both branches ("develop" and "release"), we'll always build a newly original Docker Image.
    stage('Build Docker Image') {
      when {
        anyOf {
          branch 'develop'
          branch 'release'
        }
      }
      steps {
        sh '''
          echo "Starting to build docker image"
          docker build -t ${DOCKER_REGISTRY_USERNAME}/my_obo:v1.${BUILD_NUMBER} -f Dockerfile .
        '''
      }
    }

    // In the "develop" branch, we'll use the original Docker Image and tag it with the "dev" suffix.
    stage('Push Docker Image (develop)') {
      when {
        branch 'develop'
      }
      steps {
        sh '''
          echo "Tagging the docker image"
          docker tag ${DOCKER_REGISTRY_USERNAME}/my_obo:v1.${BUILD_NUMBER} ${DOCKER_REGISTRY_USERNAME}/my_obo:v1.${BUILD_NUMBER}-dev
        '''
        sh '''
          echo "Starting to push the docker image"
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "${DOCKER_REGISTRY_USERNAME}/my_obo:v1.${BUILD_NUMBER}-dev"
        '''
      }
    }

    // In the "develop" branch, we'll directly write a new build version to another GitHub repository
    // which contains K8S manifests.
    stage('Deploy K8S Manifests (develop)') {
      when {
        branch 'develop'
      }
      steps {
        withCredentials([gitUsernamePassword(credentialsId: 'GITHUB_PERSONAL_ACCESS_TOKEN', gitToolName: 'Default')]) {
          sh '''
            rm -rf ${FINAL_PROJECT_MANIFEST_REPO_NAME}
            git clone ${FINAL_PROJECT_MANIFEST_REPO_URL}
            cd ${FINAL_PROJECT_MANIFEST_REPO_NAME}
            git checkout develop 2>/dev/null || git checkout -b develop
            git pull origin develop
          '''
        }
        script {
          sh "echo 'Updating the K8S manifests: App Deployment'"
          def app_deploy_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/app_deployment.yaml"
          def app_deploy_data = readYaml file: app_deploy_filename
          app_deploy_data.spec.template.spec.containers[0].image = "${DOCKER_REGISTRY_USERNAME}/my_obo:v1.${BUILD_NUMBER}-dev"
          sh "rm $app_deploy_filename"
          writeYaml file: app_deploy_filename, data: app_deploy_data
          sh "cat $app_deploy_filename"
          // ---
          sh "echo 'Updating the K8S manifests: App Service'"
          def app_service_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/app_service.yaml"
          def app_service_data = readYaml file: app_service_filename
          app_service_data.spec.ports[0].nodePort = "${APP_SERVICE_NODE_PORT_DEV}"
          sh "rm $app_service_filename"
          writeYaml file: app_service_filename, data: app_service_data
          sh "cat $app_service_filename"
          // ---
          sh "echo 'Updating the K8S manifests: Obo Config Database URL (Namespace: dev)'"
          def obo_config_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/obo_config.yaml"
          def obo_config_data = readYaml file: obo_config_filename
          obo_config_data.stringData.DATABASE_URL = "jdbc:mysql://mysql-service.dev.svc.cluster.local:3306/obo?useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC"
          sh "rm $obo_config_filename"
          writeYaml file: obo_config_filename, data: obo_config_data
        }
        withCredentials([gitUsernamePassword(credentialsId: 'GITHUB_PERSONAL_ACCESS_TOKEN', gitToolName: 'Default')]) {
          sh '''
            cd ${FINAL_PROJECT_MANIFEST_REPO_NAME}
            git config user.email "jenkins@example.com"
            git config user.name "Jenkins"
            git add .
            git commit -am "Update image to tag v1.${BUILD_NUMBER}-dev"
            git push origin develop
          '''
        }
      }
    }

    // In the "release" branch, we'll require the tag version input first, then write a new build version
    // to another GitHub repository which contains K8S manifests.
    stage('Deploy K8S Manifests (release)') {
      when {
        beforeInput true
        branch 'release'
      }
      input {
        message "Enter the new release version (e.g v1.2.3)"
        ok "Confirm"
        parameters {
          string(name: "IMAGE_TAG", defaultValue: "v0.0.0")
        }
      }
      steps {
        sh '''
          echo "Tagging the docker image"
          docker tag ${DOCKER_REGISTRY_USERNAME}/my_obo:v1.${BUILD_NUMBER} ${DOCKER_REGISTRY_USERNAME}/my_obo:${IMAGE_TAG}
        '''
        sh '''
          echo "Starting to push the docker image"
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "${DOCKER_REGISTRY_USERNAME}/my_obo:${IMAGE_TAG}"
        '''
        withCredentials([gitUsernamePassword(credentialsId: 'GITHUB_PERSONAL_ACCESS_TOKEN', gitToolName: 'Default')]) {
          sh '''
            rm -rf ${FINAL_PROJECT_MANIFEST_REPO_NAME}
            git clone ${FINAL_PROJECT_MANIFEST_REPO_URL}
            cd ${FINAL_PROJECT_MANIFEST_REPO_NAME}
            git checkout release 2>/dev/null || git checkout -b release
            git pull origin release
          '''
        }
        script {
          sh "echo 'Updating the K8S manifests: App Deployment'"
          def app_deploy_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/app_deployment.yaml"
          def app_deploy_data = readYaml file: app_deploy_filename
          app_deploy_data.spec.template.spec.containers[0].image = "${DOCKER_REGISTRY_USERNAME}/my_obo:${IMAGE_TAG}"
          sh "rm $app_deploy_filename"
          writeYaml file: app_deploy_filename, data: app_deploy_data
          sh "cat $app_deploy_filename"
          // ---
          sh "echo 'Updating the K8S manifests: App Service'"
          def app_service_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/app_service.yaml"
          def app_service_data = readYaml file: app_service_filename
          app_service_data.spec.ports[0].nodePort = "${APP_SERVICE_NODE_PORT_PRD}"
          sh "rm $app_service_filename"
          writeYaml file: app_service_filename, data: app_service_data
          sh "cat $app_service_filename"
          // ---
          sh "echo 'Updating the K8S manifests: Obo Config Database URL (Namespace: prd)'"
          def obo_config_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/obo_config.yaml"
          def obo_config_data = readYaml file: obo_config_filename
          obo_config_data.stringData.DATABASE_URL = "jdbc:mysql://mysql-service.prd.svc.cluster.local:3306/obo?useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC"
          sh "rm $obo_config_filename"
          writeYaml file: obo_config_filename, data: obo_config_data
        }
        withCredentials([gitUsernamePassword(credentialsId: 'GITHUB_PERSONAL_ACCESS_TOKEN', gitToolName: 'Default')]) {
          sh '''
            cd ${FINAL_PROJECT_MANIFEST_REPO_NAME}
            git config user.email "jenkins@example.com"
            git config user.name "Jenkins"
            git add .
            git commit -am "Update image to tag ${IMAGE_TAG}"
            git push origin release
          '''
        }
      }
    }

    // In both branches ("develop" and "release"), we'll clean up all unused containers, networks, images and volumes.
    stage('Cleanup Unused Docker Objects') {
      when {
        anyOf {
          branch 'develop'
          branch 'release'
        }
      }
      steps {
        sh '''
          echo "Starting to clean up all unused docker containers, networks, images, and volumes"
          docker system prune --force
        '''
      }
    }
  }
}
