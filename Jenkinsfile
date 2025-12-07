pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'oussamaabdallah/student-management'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/oussamaabdallah/student-management.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Docker Build') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .'
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline réussi !'
        }
    }
}