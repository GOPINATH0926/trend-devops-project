pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "gopinathsiva2605/trend-app:latest"
    }
    stages {
        stage('Clone') {
            steps {
                git branch: 'main', url: 'https://github.com/GOPINATH0926/trend-devops-project.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t trend-app .'
                sh "docker tag trend-app ${DOCKER_IMAGE}"
            }
        }
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
                sh 'kubectl rollout status deployment/trend-deployment'
            }
        }
    }
    post {
        success { echo 'Deployment successful!' }
        failure { echo 'Build failed. Check logs.' }
    }
}
