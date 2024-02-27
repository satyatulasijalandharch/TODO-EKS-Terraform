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
    }
}