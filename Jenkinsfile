pipeline {
  agent any
  environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
    GITHUB_USERNAME = credentials('GITHUB_USERNAME')
    FINAL_PROJECT_MANIFEST_REPO_URL = credentials('FINAL_PROJECT_MANIFEST_REPO_URL')
    FINAL_PROJECT_MANIFEST_REPO_NAME = credentials('FINAL_PROJECT_MANIFEST_REPO_NAME')
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

    // In the "develop" branch, we'll directly write a new build version to another GitHub repository which contains K8S manifests.
    stage('Deploy K8S Manifests (develop)') {
      steps {
        withCredentials([gitUsernamePassword(credentialsId: 'GITHUB_PERSONAL_ACCESS_TOKEN', gitToolName: 'Default')]) {
          sh 'rm -rf ${FINAL_PROJECT_MANIFEST_REPO_NAME}'
          sh 'git clone ${FINAL_PROJECT_MANIFEST_REPO_URL}'
        }
        script {
          sh "echo 'Updating the K8S manifests: App Deployment'"
          def app_deploy_filename = '${FINAL_PROJECT_MANIFEST_REPO_NAME}/templates/app_deployment.yaml'
          def app_deploy_data = readYaml file: app_deploy_filename
          app_deploy_data.spec.template.spec.containers[0].image = "${DOCKER_REGISTRY_USERNAME}/my_obo:v1.${BUILD_NUMBER}-dev"
          sh "rm $app_deploy_filename"
          writeYaml file: filename, data: app_deploy_data
          sh "cat $app_deploy_filename"
        }
        withCredentials([gitUsernamePassword(credentialsId: 'GITHUB_PERSONAL_ACCESS_TOKEN', gitToolName: 'Default')]) {
          sh '''
            cd ${FINAL_PROJECT_MANIFEST_REPO_NAME}
            git config user.email "jenkins@example.com"
            git config user.name "Jenkins"
            git add .
            git commit -am "Update image to tag v1.${BUILD_NUMBER}-dev"
            git push origin master
          '''
        }
      }
    }

    // In the "release" branch, we'll require the tag version input first, then write a new build version to another GitHub repository which contains K8S manifests.
    stage('Deploy to release environment') {
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

      // TODO: Update here
      steps {
        sh '''
          echo "Tag image to releae and push image"
          docker tag <dockerhub_account>/obo:v1.${BUILD_NUMBER} <dockerhub_account>/obo:${IMAGE_TAG}
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "<dockerhub_account>/obo:${IMAGE_TAG}"
        '''
        // Triển khai tới môi trường production
        script {
          sh "echo 'Deploy to kubernetes'"
          echo "Update deployment.yaml"
          def filename = 'manifests/deployment.yaml'
          def data = readYaml file: filename
          data.spec.template.spec.containers[0].image = "<dockerhub_account>/obo:${IMAGE_TAG}"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
          echo "Update service Node Port"
          def servicefile = 'manifests/service.yaml'
          def servicedata = readYaml file: servicefile
          servicedata.spec.ports[0].nodePort = 30290
          sh "rm $servicefile"
          writeYaml file: servicefile, data: servicedata
        }
        withKubeConfig([credentialsId: 'kubeconfig', serverUrl: 'https://<host_ip>:<k8s_api_server_port>']) {
          sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'
          sh 'chmod u+x ./kubectl'
          sh './kubectl apply -f manifests -n release'
        }
      }
    }
  }
}
