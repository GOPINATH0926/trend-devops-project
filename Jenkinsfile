pipeline {
    agent any

    stages {
        stage('Clone') {
            steps {
                git 'https://github.com/YOUR_USERNAME/trend-devops-project.git'
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t trend-app .'
            }
        }

        stage('Push') {
            steps {
                sh 'docker login -u USER -p PASS'
                sh 'docker tag trend-app USER/trend-app:latest'
                sh 'docker push USER/trend-app:latest'
            }
        }

        stage('Deploy') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
            }
        }
    }
}
