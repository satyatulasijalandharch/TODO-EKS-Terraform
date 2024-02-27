pipeline {
    agent any

    stages {    
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('terraform init') {
            steps {
                dir('EKS-Terraform') {
                    sh 'terraform init'
                }
            }
        }
        stage('terraform validate') {
            steps {
                dir('EKS-Terraform') {
                    sh 'terraform validate'
                }
            }
        }
        stage('terraform plan') {
            steps {
                dir('EKS-Terraform') {
                    sh 'terraform plan'
                }
            }
        }
        stage('terraform Apply/Destroy') {
            steps {
                dir('EKS-Terraform') {
                    sh 'terraform ${action} --auto-approve'
                }
            }
        }
    }
}