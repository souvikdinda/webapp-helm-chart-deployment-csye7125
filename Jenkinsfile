def webappVersion
def webappDBVersion
pipeline {
    agent any 
    
    environment {
        GCLOUD_PROJECT = 'csye7125-gke-f8c10e6f'
        GCLOUD_CLUSTER = 'my-gke-cluster'
        GCLOUD_REGION = 'us-east1'
        WEBAPP_NAMESPACE = 'webapp'
    }
    
    stages {
        stage('Authenticating with GKE') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'gke_service_account_key', variable: 'GCLOUD_SERVICE_KEY')]) {
                        sh "gcloud auth activate-service-account --key-file=${GCLOUD_SERVICE_KEY}"
                    }

                    sh "gcloud config set project ${GCLOUD_PROJECT}"
                    sh "gcloud container clusters get-credentials my-gke-cluster --region ${GCLOUD_REGION}"

                    echo "Authenticated with GKE..."

                }
            }
        }

        stage('Checking out the code') {
            steps{
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: 'main']],
                        userRemoteConfigs: [[credentialsId: 'github_token', url: 'https://github.com/csye7125-fall2023-group07/webapp-helm-chart.git']]
                    ])
                }
            }
        }

        stage('Updating values.yaml') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github_token', usernameVariable: 'GH_USERNAME', passwordVariable: 'GH_TOKEN')]) {
                        webappVersion = sh(
                            script: """
                                git ls-remote --tags https://\${GH_TOKEN}@github.com/csye7125-fall2023-group07/webapp.git |
                                awk '{print \$2}' |
                                awk -F"/" '{print \$NF}' |
                                awk -F"v" '{print \$NF}' |
                                sort -V |
                                tail -n 1
                            """,
                            returnStdout: true
                        ).trim()
                        sh "echo 'webappVersion: ${webappVersion}'"

                        webappDBVersion = sh(
                            script: """
                                git ls-remote --tags https://\${GH_TOKEN}@github.com/csye7125-fall2023-group07/webapp-db.git |
                                awk '{print \$2}' |
                                awk -F"/" '{print \$NF}' |
                                awk -F"v" '{print \$NF}' |
                                sort -V |
                                tail -n 1
                            """,
                            returnStdout: true
                        ).trim()
                        sh "echo 'webappDBVersion: ${webappDBVersion}'"
                    }

                    sh "sed -i 's/tag: [0-9].[0-9].[0-9]/tag: ${webappVersion}/' values.yaml"
                    sh "sed -i 's/flywayTag: [0-9].[0-9].[0-9]/flywayTag: ${webappDBVersion}/' values.yaml"
                }
            }
        }

        stage('Check and Create Namespace') {
            steps {
                script {
                    def namespaceExists = sh(script: "kubectl get namespace ${WEBAPP_NAMESPACE}", returnStatus: true)
                    
                    if (namespaceExists != 0) {
                        sh "kubectl create namespace ${WEBAPP_NAMESPACE}"
                        echo "Namespace ${WEBAPP_NAMESPACE} created."
                    } else {
                        echo "Namespace ${WEBAPP_NAMESPACE} already exists."
                    }
                }
            }
        }
        
        stage('Label Namespace for Istio Injection') {
            steps {
                script {
                    sh "kubectl label namespace ${WEBAPP_NAMESPACE} istio-injection=enabled --overwrite"
                    echo "Namespace ${WEBAPP_NAMESPACE} labeled for Istio injection."
                }
            }
        }

        stage('Deploying to GKE') {
            steps {
                sh 'helm upgrade --install webapp-chart -n ${WEBAPP_NAMESPACE} .'
            }
        }

        stage('Revoke GKE access') {
            steps {
                sh 'gcloud auth revoke --all'
            }
            
        }
    }
}
