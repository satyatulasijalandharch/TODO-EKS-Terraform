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

        stage('Build Started') {when {
                expression { params.action != 'destroy' }
            }
            steps {
                slackSend(
                    botUser: true,
                    channel: SLACK_CHANNEL,
                    color: 'good',
                    message: "Kubernetes Build [${env.BUILD_NUMBER}] Started",
                    teamDomain: 'DevOps',
                    tokenCredentialId: SLACK_CREDENTIAL_ID
                )
            }
        }
        stage('Destroy Started') {
            when {
                expression { params.action == 'destroy' }
            }
            steps {
                slackSend(
                    botUser: true,
                    channel: SLACK_CHANNEL,
                    color: 'good',
                    message: "Kubernetes Destroy [${env.BUILD_NUMBER}] Started",
                    teamDomain: 'DevOps',
                    tokenCredentialId: SLACK_CREDENTIAL_ID
                )
            }
        }

        stage('Cluster Cleanup') {
            when {
                expression { params.action == 'destroy' }
            }
            steps {
                script {
                    if (sh(script: "aws eks --region ap-south-1 describe-cluster --name EKS_TODO", returnStatus: true) == 0) {
                        sh "aws eks --region ap-south-1 update-kubeconfig --name EKS_TODO"
                    }
                    
                    // Check if ArgoCD exists before uninstalling
                    if (sh(script: "kubectl get ns ${ARGOCD_NAMESPACE}", returnStatus: true) == 0) {
                        sh "kubectl delete -n ${ARGOCD_NAMESPACE} -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                        sh "kubectl wait --for=delete pod -l app.kubernetes.io/name=argocd-server -n ${ARGOCD_NAMESPACE}"
                    }
                    
                    // Check if Ingress-Nginx exists before uninstalling
                    if (sh(script: "kubectl get ns ${INGRESS_NAMESPACE}", returnStatus: true) == 0) {
                        sh "kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/aws/deploy.yaml -n ${INGRESS_NAMESPACE}"
                        sh "kubectl wait --for=delete pod -l app.kubernetes.io/component=controller -n ${INGRESS_NAMESPACE}"
                    }

                    echo "Cluster cleanup completed."
                }
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
                sh 'terraform ${action} --auto-approve'
            }
        }


        stage('Install Ingress-Nginx') {
            when {
                expression { params.action != 'destroy' }
            }
            steps {
                script {
                    sh "aws eks --region ap-south-1 update-kubeconfig --name EKS_TODO"

                    def namespaceExists = sh(script: "kubectl get namespace ${INGRESS_NAMESPACE}", returnStatus: true) == 0

                    if (!namespaceExists) {
                        sh "kubectl create namespace ${INGRESS_NAMESPACE}"
                        sh "kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/aws/deploy.yaml -n ${INGRESS_NAMESPACE}"
                        sh "kubectl wait --for=condition=Ready pod -l app.kubernetes.io/component=controller -n ${INGRESS_NAMESPACE} --timeout=2m"
                    } else {
                        echo "Namespace ${INGRESS_NAMESPACE} already exists. Skipping namespace creation."
                    }                    
                }
            }
        }
        stage('Install ArgoCD') {
            when {
                expression { params.action != 'destroy' }
            }
            steps {
                script {
                    sh "aws eks --region ap-south-1 update-kubeconfig --name EKS_TODO"

                    def namespaceExists = sh(script: "kubectl get namespace ${ARGOCD_NAMESPACE}", returnStatus: true) == 0

                    if (!namespaceExists) {
                        sh "kubectl create namespace ${ARGOCD_NAMESPACE}"
                        sh "kubectl apply -n ${ARGOCD_NAMESPACE} -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                        sh "kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=argocd-server -n ${ARGOCD_NAMESPACE} --timeout=2m"
                        sh 'kubectl patch svc argocd-server -n argocd -p {\'"spec": {"type": "NodePort"}\'}'
                        //kubectl patch svc argocd-server -n argocd --type='json' -p='[{"op": "replace", "path": "/spec/type", "value": "NodePort"}]'
                        def argocdPassword = sh(script: "kubectl -n ${ARGOCD_NAMESPACE} get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode", returnStdout: true).trim()
                        echo "ArgoCD Admin Password: ${argocdPassword}"
                        sh "kubectl apply -f argocd-ingress.yaml"
                        sh "sleep 1m"
                        def ingressOutput = sh(script: 'kubectl get ingress -n argocd', returnStdout: true).trim()
                        echo "Ingress Output:\n${ingressOutput}"
                        sh "aws eks --region ap-south-1 update-kubeconfig --name EKS_TODO"
                    } else {
                        echo "Namespace ${ARGOCD_NAMESPACE} already exists. Skipping namespace creation."
                    }                    
                }
            }
        }
    }
    post {
        success {
            script {
                def attachments = [
                    [
                        "color": "good",
                        "text": "Build Successful",
                        "fields": [
                            ["title": "Project", "value": "${env.JOB_NAME}", "short": true],
                            ["title": "Build Number", "value": "${env.BUILD_NUMBER}", "short": true],
                            ["title": "URL", "value": "${env.BUILD_URL}", "short": false]
                        ]
                    ]
                ]
                slackSend color: 'good', channel: '#devops', attachments: attachments
            }
        }
        failure {
            script {
                def attachments = [
                    [
                        "color": "danger",
                        "text": "Build Failed",
                        "fields": [
                            ["title": "Project", "value": "${env.JOB_NAME}", "short": true],
                            ["title": "Build Number", "value": "${env.BUILD_NUMBER}", "short": true],
                            ["title": "URL", "value": "${env.BUILD_URL}", "short": false]
                        ]
                    ]
                ]
                slackSend color: 'danger', channel: '#devops', attachments: attachments

                // Add Terraform destroy commands here
                if (sh(script: "aws eks --region ap-south-1 describe-cluster --name EKS_TODO", returnStatus: true) == 0) {
                    sh "aws eks --region ap-south-1 update-kubeconfig --name EKS_TODO"
                }
                // Check if ArgoCD exists before uninstalling
                if (sh(script: "kubectl get ns ${ARGOCD_NAMESPACE}", returnStatus: true) == 0) {
                    sh "kubectl delete -n ${ARGOCD_NAMESPACE} -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                    sh "kubectl wait --for=delete pod -l app.kubernetes.io/name=argocd-server -n ${ARGOCD_NAMESPACE}"
                }
                
                // Check if Ingress-Nginx exists before uninstalling
                if (sh(script: "kubectl get ns ${INGRESS_NAMESPACE}", returnStatus: true) == 0) {
                    sh "kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/aws/deploy.yaml -n ${INGRESS_NAMESPACE}"
                    sh "kubectl wait --for=delete pod -l app.kubernetes.io/component=controller -n ${INGRESS_NAMESPACE}"
                }
                echo "Cluster cleanup completed."
                sh "terraform init"
                sh "terraform destroy -auto-approve"

                // Send notification about Terraform destroy
                slackSend(
                    color: 'warning',
                    channel: '#devops',
                    message: "Terraform infrastructure destroyed due to build failure"
                )
            }
        }
    }
}
