pipeline{
    agent any
    stages{
        stage('Clone Repository'){
            steps{
                git branch: 'main', url:'https://github.com/cryo37/DevOps-Two-Tier-Flask-App.git'
            }
        }
        stage('Build Docker Image'){
            steps{
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker'){
            steps{
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }
    }
}
