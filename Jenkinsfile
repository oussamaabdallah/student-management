pipeline {
    agent any
    
    environment {
        // CHANGEZ ICI : Remplacez par votre username Docker Hub
        DOCKERHUB_USER = 'oussamaabdallah'
        DOCKER_IMAGE_NAME = 'student-management'
        DOCKER_IMAGE = "${DOCKERHUB_USER}/${DOCKER_IMAGE_NAME}"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
    }
    
    tools {
        maven 'Maven3'  // Doit être configuré dans Jenkins
    }
    
    stages {
        // ÉTAPE 1 : Récupération du code
        stage('Checkout from GitHub') {
            steps {
                echo '📥 Clonage du dépôt GitHub...'
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/oussamaabdallah/student-management.git',
                        credentialsId: ''  // Optionnel si repo public
                    ]]
                ])
                
                // Alternative simple :
                // git branch: 'main', 
                //     url: 'https://github.com/oussamaabdallah/student-management.git'
            }
        }
        
        // ÉTAPE 2 : Build Maven
        stage('Maven Build') {
            steps {
                echo '🔨 Construction du projet avec Maven...'
                sh '''
                    mvn clean compile
                    mvn package -DskipTests
                '''
                
                // Vérification
                sh 'ls -la target/*.jar'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
        
        // ÉTAPE 3 : Tests unitaires (optionnel)
        stage('Unit Tests') {
            steps {
                echo '🧪 Exécution des tests unitaires...'
                sh 'mvn test'
                
                // Rapport de tests
                junit 'target/surefire-reports/*.xml'
            }
        }
        
        // ÉTAPE 4 : Analyse SonarQube (optionnel)
        stage('SonarQube Analysis') {
            when {
                environment name: 'RUN_SONARQUBE', value: 'true'
            }
            steps {
                echo '📊 Analyse de la qualité du code...'
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        // ÉTAPE 5 : Construction image Docker
        stage('Build Docker Image') {
            steps {
                echo '🐳 Construction de l\'image Docker...'
                script {
                    // Build avec tag
                    sh """
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                    """
                    
                    // Lister les images
                    sh 'docker images | grep student-management'
                }
            }
        }
        
        // ÉTAPE 6 : Scan de sécurité (optionnel)
        stage('Security Scan') {
            steps {
                echo '🔒 Scan de sécurité de l\'image...'
                script {
                    sh "docker scan ${DOCKER_IMAGE}:${DOCKER_TAG} --file Dockerfile"
                }
            }
        }
        
        // ÉTAPE 7 : Push vers Docker Hub
        stage('Push to Docker Hub') {
            steps {
                echo '⬆️  Pushing image to Docker Hub...'
                script {
                    // Utilisation des credentials Docker Hub
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',  // ID dans Jenkins
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        // Connexion à Docker Hub
                        sh """
                            echo "Connexion à Docker Hub..."
                            echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                        """
                        
                        // Push des images
                        sh """
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                            echo "✅ Images poussées avec succès!"
                        """
                        
                        // Déconnexion
                        sh 'docker logout'
                    }
                }
            }
        }
        
        // ÉTAPE 8 : Déploiement de test (optionnel)
        stage('Deploy to Test Environment') {
            steps {
                echo '🚀 Déploiement en environnement de test...'
                script {
                    sh """
                        # Arrêter l'ancien conteneur si existe
                        docker stop student-app-test 2>/dev/null || true
                        docker rm student-app-test 2>/dev/null || true
                        
                        # Lancer le nouveau conteneur
                        docker run -d \\
                            -p 9090:8089 \\
                            --name student-app-test \\
                            --restart unless-stopped \\
                            ${DOCKER_IMAGE}:latest
                        
                        # Attendre le démarrage
                        sleep 15
                        
                        # Tester l'application
                        echo "Test de l'application déployée..."
                        curl -f http://localhost:9090/ || exit 1
                        echo "✅ Application déployée avec succès!"
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ SUCCÈS : Pipeline terminé avec succès !'
            echo "📦 Image Docker : ${DOCKER_IMAGE}:${DOCKER_TAG}"
            echo "🔗 Docker Hub : https://hub.docker.com/r/${DOCKERHUB_USER}/${DOCKER_IMAGE_NAME}"
            echo "🌐 Application de test : http://localhost:9090"
            
            // Notification email (optionnel)
            emailext (
                subject: "SUCCÈS : Build #${BUILD_NUMBER} - Student Management",
                body: """
                    Le pipeline Jenkins a réussi.
                    
                    Détails :
                    - Job : ${env.JOB_NAME}
                    - Build : #${env.BUILD_NUMBER}
                    - Image Docker : ${DOCKER_IMAGE}:${DOCKER_TAG}
                    - Docker Hub : https://hub.docker.com/r/${DOCKERHUB_USER}/${DOCKER_IMAGE_NAME}
                    
                    Cordialement,
                    Jenkins
                """,
                to: 'dev-team@example.com',  // Changez cette adresse
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        
        failure {
            echo '❌ ÉCHEC : Le pipeline a échoué.'
            echo '📋 Consultez les logs pour plus de détails.'
            
            // Notification d'erreur
            emailext (
                subject: "ÉCHEC : Build #${BUILD_NUMBER} - Student Management",
                body: "Le pipeline Jenkins a échoué. Vérifiez les logs.",
                to: 'dev-team@example.com',
                recipientProviders: [[$class: 'RequesterRecipientProvider']]
            )
        }
        
        unstable {
            echo '⚠️  INSTABLE : Le pipeline est instable (tests échoués).'
        }
        
        always {
            echo '🧹 Nettoyage des ressources...'
            script {
                // Nettoyage Docker
                sh '''
                    # Arrêter les conteneurs de test
                    docker stop student-app-test 2>/dev/null || true
                    docker rm student-app-test 2>/dev/null || true
                    
                    # Nettoyer les images intermédiaires
                    docker image prune -f 2>/dev/null || true
                    docker container prune -f 2>/dev/null || true
                '''
                
                // Sauvegarde des artefacts
                archiveArtifacts artifacts: '**/target/*.jar, **/Dockerfile, **/pom.xml', fingerprint: true
                
                // Rapport de couverture de tests
                publishHTML([
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'target/site/jacoco',
                    reportFiles: 'index.html',
                    reportName: 'Test Coverage Report'
                ])
            }
        }
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        retry(2)
    }
    
    parameters {
        booleanParam(
            name: 'RUN_SONARQUBE',
            defaultValue: false,
            description: 'Exécuter l\'analyse SonarQube'
        )
        
        choice(
            name: 'DEPLOY_ENV',
            choices: ['none', 'test', 'staging'],
            description: 'Environnement de déploiement'
        )
        
        string(
            name: 'CUSTOM_TAG',
            defaultValue: '',
            description: 'Tag personnalisé pour l\'image Docker'
        )
    }
}