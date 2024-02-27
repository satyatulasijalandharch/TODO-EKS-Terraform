pipeline {
    agent any

    environment {
        INGRESS_NAMESPACE = 'ingress-nginx'
        ARGOCD_NAMESPACE = 'argocd'
        SLACK_CHANNEL = '#devops'
        SLACK_CREDENTIAL_ID = 'slack'
    }

    stages {
        // Stage 1: Checkout Source Code
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Stage 2: Notify Build Started
        stage('Started') {
            steps {
                slackSend botUser: true, channel: SLACK_CHANNEL, color: 'good', message: 'Kubernetes Build Started', teamDomain: 'DevOps', tokenCredentialId: SLACK_CREDENTIAL_ID
            }
        }

        // Stage 3: Terraform Initialization
        stage('terraform init') {
            steps {
                sh 'terraform init'
            }
        }

        // Stage 4: Terraform Validation
        stage('terraform validate') {
            steps {
                sh 'terraform validate'
            }
        }

        // Stage 5: Terraform Plan
        stage('terraform plan') {
            steps {
                sh 'terraform plan'
            }
        }

        // Stage 6: Terraform Apply/Destroy
        stage('terraform Apply/Destroy') {
            steps {
                sh 'terraform apply --auto-approve'
            }
        }

        // Stage 7: Verify Cluster Provisioning
        stage('Verify Cluster Provisioning') {
            steps {
                script {
                    def clusterReady = false
                    def maxRetries = 30
                    def retryInterval = 30

                    for (int i = 0; i < maxRetries; i++) {
                        // Check if nodes are ready
                        def nodeStatus = sh(script: 'kubectl get nodes --field-selector=status.phase=Running --no-headers', returnStatus: true).trim()

                        if (nodeStatus == 0) {
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

        // Stage 8: Install Ingress-Nginx
        stage('Install Ingress-Nginx') {
            steps {
                script {
                    // Install Ingress-Nginx in a separate namespace
                    sh "kubectl create namespace ${INGRESS_NAMESPACE}"
                    sh "kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml -n ${INGRESS_NAMESPACE}"

                    // Wait for Ingress-Nginx pods to be ready
                    sh "kubectl wait --for=condition=Ready pod -l app.kubernetes.io/component=controller -n ${INGRESS_NAMESPACE}"
                }
            }
        }

        // Stage 9: Install ArgoCD
        stage('Install ArgoCD') {
            steps {
                script {
                    // Install ArgoCD in a separate namespace
                    sh "kubectl create namespace ${ARGOCD_NAMESPACE}"
                    sh "kubectl apply -n ${ARGOCD_NAMESPACE} -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"

                    // Wait for ArgoCD pods to be ready
                    sh "kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=argocd-server -n ${ARGOCD_NAMESPACE}"

                    // Extract ArgoCD admin password
                    def argocdPassword = sh(script: "kubectl -n ${ARGOCD_NAMESPACE} get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode", returnStdout: true).trim()

                    // Print ArgoCD admin password
                    echo "ArgoCD Admin Password: ${argocdPassword}"

                    // Configure Ingress for ArgoCD in the ArgoCD namespace
                    sh "kubectl apply -f argocd-ingress.yaml -n ${ARGOCD_NAMESPACE}"
                }
            }
        }
    }

    // Stage 10: Notify Pipeline Status
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
