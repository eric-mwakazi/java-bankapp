pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }

    environment {
        IMAGE_NAME = "mwakazi/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME = tool 'sonar-scanner'
        ENGINEER_MAIL = 'mwakazieric@gmail.com'
    }

    stages {
        stage('Initialize') {
            stages {
                stage('Git Checkout') {
                    steps {
                        git branch: 'main', credentialsId: 'gitcred-blue-green-cicd', url: 'http://192.168.1.13/mwakazi/java-bankapp.git'
                    }
                }
            }
        }

        stage('Build & Test') {
            stages {
                stage('Build Java Project') {
                    steps {
                        withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', traceability: true) {
                            sh 'mvn package -DskipTests=true'
                        }
                    }
                }
                stage('Publish Artifact To Nexus') {
                    steps {
                        withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', traceability: true) {
                            sh 'mvn deploy -DskipTests=true'
                        }
                    }
                }
            }
        }

        stage('Code Quality & Security') {
            stages {
                stage('SonarQube Analysis') {
                    steps {
                        withSonarQubeEnv('sonar') {
                            sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=javabankapp -Dsonar.projectName=javabankapp -Dsonar.java.binaries=target/classes"
                        }
                    }
                }
                stage('SonarQube Quality Gate') {
                    steps {
                        timeout(time: 1, unit: 'MINUTES') {
                            script {
                                def qg = waitForQualityGate()
                                if (qg.status != 'OK') {
                                    error "Pipeline failed due to quality gate: ${qg.status}"
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-blue-green-cicd') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                    emailext subject: "Docker Push Successful: ${IMAGE_NAME}:${TAG}",
                        body: "The Docker image ${IMAGE_NAME}:${TAG} has been pushed successfully.",
                        to: "${ENGINEER_MAIL}"
                }
            }
        }

        stage('Ensure Namespace Exists') {
            steps {
                ensureNamespace()
            }
        }
        stage('Deploy Database') {
            steps {
                deployToKubernetes('mysql-ds.yml')
            }
        }

        stage('Deploy App') {
            steps {
                script {
                    def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
                    deployToKubernetes(deploymentFile)
                }
            }
        }

        stage('Deploy Service') {
            steps {
                deployToKubernetes('bankapp-service.yml')
            }
        }

        stage('Manage Traffic') {
            steps {
                script {
                    if (params.SWITCH_TRAFFIC) {
                        recreateService(params.DEPLOY_ENV)
                    }
                }
            }
        }

        stage('Verification') {
            steps {
                verifyDeployment(params.DEPLOY_ENV)
                emailext subject: "Deployment Verification: ${IMAGE_NAME}:${TAG}",
                    body: "Deployment verification completed for ${IMAGE_NAME}:${TAG}.",
                    to: "${ENGINEER_MAIL}"
            }
        }
    }

    post {
        always {
            emailext subject: "Pipeline Result: ${currentBuild.currentResult}",
                body: "Build ${currentBuild.fullDisplayName} result: ${currentBuild.currentResult}",
                to: "${ENGINEER_MAIL}"
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed. Investigate immediately!'
        }
    }
}

def deployToKubernetes(file) {
    script {
        withKubeConfig(
            caCertificate: '',
            clusterName: 'local-cluster',
            credentialsId: 'k8-token',
            namespace: 'webapps',
            serverUrl: 'https://192.168.1.81:6443'
        ) {
            sh "kubectl apply -f ${file} -n ${KUBE_NAMESPACE}"
        }
    }
}

def ensureNamespace() {
    script {
        withKubeConfig(
            caCertificate: '',
            clusterName: 'local-cluster',
            credentialsId: 'k8-token',
            namespace: 'webapps',
            serverUrl: 'https://192.168.1.81:6443'
        ) {
            sh "kubectl get namespace ${KUBE_NAMESPACE} || kubectl create namespace ${KUBE_NAMESPACE}"
        }
    }
}

def recreateService(newEnv) {
    script {
        withKubeConfig(
            caCertificate: '',
            clusterName: 'local-cluster',
            credentialsId: 'k8-token',
            namespace: 'webapps',
            serverUrl: 'https://192.168.1.81:6443'
        ) {
            sh "kubectl delete svc bankapp-service -n ${KUBE_NAMESPACE} || true"
            sh "kubectl expose deployment bankapp-${newEnv} --port=80 --target-port=8080 --name=bankapp-service -n ${KUBE_NAMESPACE} --type=NodePort"
        }
        echo "Traffic has been switched to the ${newEnv} environment."
    }
}

def verifyDeployment(verifyEnv) {
    script {
        withKubeConfig(
            caCertificate: '',
            clusterName: 'local-cluster',
            credentialsId: 'k8-token',
            namespace: 'webapps',
            serverUrl: 'https://192.168.1.81:6443'
        ) {
            sh """
                kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
            """
        }
    }
}