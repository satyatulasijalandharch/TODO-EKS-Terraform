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
            }
        }
    }
}
