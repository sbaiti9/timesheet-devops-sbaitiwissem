pipeline {
    agent any
    
    environment {
        GIT_REPO = 'https://github.com/sbaiti9/timesheet-devops-sbaitiwissem.git'
        GIT_BRANCH = 'main'
        SONAR_HOST_URL = 'http://192.168.33.10:9000'
        PROJECT_DIR = 'TP-Projet-2025'
        MAVEN_HOME = '/usr/share/maven'
        
        // Configuration Docker Hub
        DOCKER_HUB_CREDENTIALS = 'dockerhub'
        DOCKER_IMAGE_NAME = 'wissemsbaiti/timesheet-app'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        
        stage('1. Récupération du code depuis Git') {
            steps {
                echo 'Clonage du repository Git...'
                git branch: "${GIT_BRANCH}", 
                    credentialsId: 'github-credentials', 
                    url: "${GIT_REPO}"
                echo 'Code récupéré avec succès'
            }
        }
        
        stage('2. Nettoyage et Compilation') {
            steps {
                echo 'Nettoyage et compilation du projet Maven...'
                dir("${PROJECT_DIR}") {
                    sh 'mvn clean compile -DskipTests'
                }
                echo 'Nettoyage et compilation terminés'
            }
        }
        
        stage('3. Analyse SonarQube') {
            steps {
                echo '🔍 Analyse de la qualité du code avec SonarQube...'
                dir("${PROJECT_DIR}") {
                    withSonarQubeEnv('sonarqube') {
                        sh """
                            mvn sonar:sonar \
                              -Dsonar.projectKey=timesheet-app \
                              -Dsonar.host.url=${SONAR_HOST_URL}
                        """
                    }
                }
                echo 'Analyse SonarQube terminée'
            }
        }
        
        stage('4. Génération du fichier JAR') {
            steps {
                echo 'Génération du fichier JAR...'
                dir("${PROJECT_DIR}") {
                    sh 'mvn package -DskipTests'
                }
                echo 'Fichier JAR généré avec succès'
            }
        }
        
        stage('5. Construction de l\'image Docker') {
            steps {
                echo 'Construction de l\'image Docker...'
                dir("${PROJECT_DIR}") {
                    script {
                        sh """
                            docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} .
                            docker tag ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ${DOCKER_IMAGE_NAME}:latest
                        """
                    }
                }
                echo 'Image Docker construite avec succès'
            }
        }
        
        stage('6. Push de l\'image sur Docker Hub') {
            steps {
                echo 'Connexion à Docker Hub et push de l\'image...'
                script {
                    withCredentials([usernamePassword(
                        credentialsId: "${DOCKER_HUB_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \${DOCKER_PASS} | docker login -u \${DOCKER_USER} --password-stdin
                            docker push ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                            docker push ${DOCKER_IMAGE_NAME}:latest
                            docker logout
                        """
                    }
                }
                echo 'Image Docker poussée sur Docker Hub avec succès'
            }
        }
        
        stage('7. Déploiement MySQL sur Kubernetes') {
            steps {
                echo 'Déploiement de MySQL sur le cluster Kubernetes...'
                script {
                    sh """
                        kubectl apply -f k8s/mysql-deployment.yaml
                        echo 'Attente du démarrage de MySQL...'
                        kubectl wait --for=condition=ready pod -l app=mysql -n devops --timeout=600s
                    """
                }
                echo 'MySQL déployé avec succès'
            }
        }
        
        stage('8. Déploiement Spring Boot sur Kubernetes') {
            steps {
                echo 'Déploiement de l\'application Spring Boot sur Kubernetes...'
                script {
                    sh """
                        kubectl apply -f k8s/spring-deployment.yaml
                        kubectl set image deployment/spring-app spring-app=${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} -n devops
                        kubectl rollout status deployment/spring-app -n devops --timeout=300s
                    """
                }
                echo 'Application Spring Boot déployée avec succès'
            }
        }
        
        stage('9. Vérification du déploiement') {
            steps {
                echo 'Vérification du déploiement...'
                script {
                    sh """
                        echo '=== PODS ==='
                        kubectl get pods -n devops
                        
                        echo '=== SERVICES ==='
                        kubectl get svc -n devops
                        
                        echo '=== URL ACCES ==='
                        echo 'Service accessible via NodePort sur le port 30080'
                        NODE_IP=\$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
                        echo "URL: http://\${NODE_IP}:30080"
                    """
                }
                echo 'Vérification terminée'
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline exécuté avec succès !'
            echo "Image Docker: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
            echo "Application déployée sur Kubernetes dans le namespace 'devops'"
        }
        failure {
            echo '❌ Le pipeline a échoué. Vérifiez les logs ci-dessus.'
        }
        always {
            echo 'Nettoyage des images Docker locales...'
            sh """
                docker rmi ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} || true
                docker rmi ${DOCKER_IMAGE_NAME}:latest || true
            """
        }
    }
}