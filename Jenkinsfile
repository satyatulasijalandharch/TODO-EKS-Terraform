pipeline {
    agent any

    environment {
        INGRESS_NAMESPACE = 'ingress-nginx'
        ARGOCD_NAMESPACE = 'argocd'
        SLACK_CHANNEL = '#devops'
        SLACK_CREDENTIAL_ID = 'slack'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Started') {
            steps {
                slackSend botUser: true, channel: SLACK_CHANNEL, color: 'good', message: 'Kubernetes Build Started', teamDomain: 'DevOps', tokenCredentialId: SLACK_CREDENTIAL_ID
            }
        }

        stage('terraform init') {
            steps {
                sh 'terraform init'
            }
        }

        stage('terraform validate') {
            steps {
                sh 'terraform validate'
            }
        }

        stage('terraform plan') {
            steps {
                sh 'terraform plan'
            }
        }

        stage('terraform Apply/Destroy') {
            steps {
                sh 'terraform apply --auto-approve'
            }
        }

        stage('Verify Cluster Provisioning') {
            steps {
                script {
                    def clusterReady = false
                    def maxRetries = 30
                    def retryInterval = 30

                    for (int i = 0; i < maxRetries; i++) {
                        def nodeStatus = sh(script: 'kubectl get nodes --selector=status.phase=Running --no-headers', returnStatus: true).trim()

                        if (nodeStatus == 0) {
                            // Change the condition to check for non-zero exit code
                            clusterReady = true
                            break
                        } else {
                            echo "Cluster not ready. Retrying in ${retryInterval} seconds..."
                            sleep retryInterval
                        }
                    }

                    if (!clusterReady) {
                        error "Cluster provisioning did not complete within the expected time."
                    }
                }
            }
        }stage('Verify Cluster Provisioning') {
            steps {
                script {
                    def clusterReady = false
                    def maxRetries = 30
                    def retryInterval = 30

                    for (int i = 0; i < maxRetries; i++) {
                        def nodeStatus = sh(script: 'kubectl get nodes --selector=status.phase=Running --no-headers', returnStatus: true).trim()

                        if (nodeStatus.isEmpty()) {
                            clusterReady = true
                            break
                        } else {
                            echo "Cluster not ready. Retrying in ${retryInterval} seconds..."
                            sleep retryInterval
                        }
                    }

                    if (!clusterReady) {
                        error "Cluster provisioning did not complete within the expected time."
                    }
                }
            }
        }

        stage('Install Ingress-Nginx') {
            steps {
                script {
                    sh "aws eks update-kubeconfig --name EKS_TODO --region ap-south-1"
                    sh "kubectl create namespace ${INGRESS_NAMESPACE}"
                    sh "kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml -n ${INGRESS_NAMESPACE}"
                    sh "kubectl wait --for=condition=Ready pod -l app.kubernetes.io/component=controller -n ${INGRESS_NAMESPACE}"
                }
            }
        }

        stage('Install ArgoCD') {
            steps {
                script {
                    sh "kubectl create namespace ${ARGOCD_NAMESPACE}"
                    sh "kubectl apply -n ${ARGOCD_NAMESPACE} -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                    sh "kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=argocd-server -n ${ARGOCD_NAMESPACE}"

                    def argocdPassword = sh(script: "kubectl -n ${ARGOCD_NAMESPACE} get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode", returnStdout: true).trim()
                    echo "ArgoCD Admin Password: ${argocdPassword}"

                    sh "kubectl apply -f argocd-ingress.yaml -n ${ARGOCD_NAMESPACE}"
                }
            }
        }

        stage('Notify Pipeline Status') {
            steps {
                script {
                    currentBuild.result = currentBuild.resultIsBetterOrEqualTo('FAILURE') ? 'FAILURE' : 'SUCCESS'

                    slackSend color: currentBuild.result == 'SUCCESS' ? 'good' : 'danger',
                            message: "Kubernetes Build ${currentBuild.result}",
                            teamDomain: 'DevOps',
                            tokenCredentialId: SLACK_CREDENTIAL_ID,
                            channel: SLACK_CHANNEL
                }
            }
        }
    }
}
