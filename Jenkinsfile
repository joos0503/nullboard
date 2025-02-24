pipeline {
    agent any

    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to build from')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Tag for the Docker image')
        string(name: 'DOCKER_REGISTRY', defaultValue: '192.168.86.32:5000', description: 'Docker registry')
        string(name: 'DOCKER_REGISTRY_URL', defaultValue: 'http://192.168.86.32:5000', description: 'Docker registry url')
        string(name: 'DOCKER_CREDENTIALS', defaultValue: 'local-registry-credentials', description: 'Docker registry credentials')
    }
    environment {
        APP_NAME = 'nullboard'
        APP_IMAGE = "${params.DOCKER_REGISTRY}/nullboard"
        DOCKER_REGISTRY_URL = "${params.DOCKER_REGISTRY_URL}"
        DOCKER_CREDENTIALS = "${params.DOCKER_CREDENTIALS}"
        K8S_NAMESPACE = 'default'
        KUBECONFIG_CREDENTIALS = 'k8s-token-file'
    }

    stages {
        stage('Prepare Workspace') {
            steps {
                cleanWorkspace()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: "${params.BRANCH}", credentialsId: 'github-ssh-key-2', url: 'git@github.com:joos0503/nullboard.git'
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                buildAndPushDockerImage("${APP_IMAGE}", "${params.IMAGE_TAG}")
            }
        }

        stage('Clean Up Previous Deployment') {
            steps {
                cleanPreviousDeployment("${APP_NAME}", "${K8S_NAMESPACE}")
            }
        }

        stage('Create Docker Registry Secret in Kubernetes') {
            steps {
                createDockerRegistrySecret()
            }
        }

        // stage('Deploy to Kubernetes via Helm') {
        //     steps {
        //         echo "Deploying to Kubernetes namespace: ${K8S_NAMESPACE}"
        //         deployToKubernetesViaHelm("${APP_NAME}", "${APP_IMAGE}", "${params.IMAGE_TAG}", "${K8S_NAMESPACE}")
        //     }
        // }
    }

    post {
        always {
            cleanWorkspace()
        }
        success {
            echo 'Build and Deployment completed successfully!'
        }
        failure {
            echo 'Build or Deployment failed.'
        }
    }
}

def cleanWorkspace() {
    deleteDir()  // Clean the workspace to ensure no leftover files
}

def buildAndPushDockerImage(appImage, imageTag) {
    script {
        docker.withRegistry(DOCKER_REGISTRY_URL, DOCKER_CREDENTIALS) {
            def builtImage = docker.build("${appImage}:${imageTag}", '.')
            builtImage.push()
        }
    }
}

def cleanPreviousDeployment(appName, namespace) {
    script {
        sh """
        microk8s kubectl delete deployment ${appName} -n ${namespace} || true
        microk8s kubectl delete svc ${appName} -n ${namespace} || true
        """
    }
}

def createDockerRegistrySecret() {
    script {
        withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh """
            microk8s kubectl create secret docker-registry regcred-local \
            --docker-server=${DOCKER_REGISTRY_URL} \
            --docker-username=${DOCKER_USER} \
            --docker-password=${DOCKER_PASS} \
            --docker-email=johanosterberg@hotmail.com --dry-run=client -o yaml | microk8s kubectl apply -f -
            """
        }
    }
}

def deployToKubernetesViaHelm(appName, appImage, imageTag, namespace) {
    script {
            withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG')]) {
                sh """
                microk8s kubectl create namespace ${namespace} || true
            microk8s helm3 upgrade --install ${appName} ./helm \
            --namespace ${namespace} \
            --set image.repository=${appImage} \
            --set image.tag=${imageTag} \
            --values helm/values.yaml
            """
            }
    }
}
