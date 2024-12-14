pipeline {
  agent any
  environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
    GITHUB_USERNAME = credentials('GITHUB_USERNAME')
    FINAL_PROJECT_MANIFEST_REPO_URL = "https://github.com/1653072/techmaster-final-project-obo-manifest.git"
    FINAL_PROJECT_MANIFEST_REPO_NAME = "techmaster-final-project-obo-manifest"
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
            git checkout develop 2>/dev/null || git checkout -b develop
          '''
        }
        script {
          sh "echo 'Updating the K8S manifests: App Deployment'"
          def app_deploy_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/app_deployment.yaml"
          def app_deploy_data = readYaml file: app_deploy_filename
          app_deploy_data.spec.template.spec.containers[0].image = "${DOCKER_REGISTRY_USERNAME}/my_obo:v1.${BUILD_NUMBER}-dev"
          sh "rm $app_deploy_filename"
          writeYaml file: filename, data: app_deploy_data
          sh "cat $app_deploy_filename"
          // ---
          sh "echo 'Updating the K8S manifests: Obo Config Database URL (Namespace: dev)'"
          def obo_config_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/obo_config.yaml"
          def obo_config_data = readYaml file: obo_config_filename
          obo_config_data.stringData.DATABASE_URL = "jdbc:mysql://mysql-service.dev.svc.cluster.local:3306/obo?useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC"
          sh "rm $obo_config_filename"
          writeYaml file: filename, data: obo_config_data
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
            git checkout release 2>/dev/null || git checkout -b release
          '''
        }
        script {
          sh "echo 'Updating the K8S manifests: App Deployment'"
          def app_deploy_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/app_deployment.yaml"
          def app_deploy_data = readYaml file: app_deploy_filename
          app_deploy_data.spec.template.spec.containers[0].image = "${DOCKER_REGISTRY_USERNAME}/my_obo:${IMAGE_TAG}"
          sh "rm $app_deploy_filename"
          writeYaml file: filename, data: app_deploy_data
          sh "cat $app_deploy_filename"
          // ---
          sh "echo 'Updating the K8S manifests: Obo Config Database URL (Namespace: prd)'"
          def obo_config_filename = "${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/obo_config.yaml"
          def obo_config_data = readYaml file: obo_config_filename
          obo_config_data.stringData.DATABASE_URL = "jdbc:mysql://mysql-service.prd.svc.cluster.local:3306/obo?useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC"
          sh "rm $obo_config_filename"
          writeYaml file: filename, data: obo_config_data
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
  }
}
