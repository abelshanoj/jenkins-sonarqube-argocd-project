pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '--network devops -v /var/run/docker.sock:/var/run/docker.sock -u root'
        }
    }

    environment {
        DOCKER_IMAGE = "abelshanoj/spring-boot-app:${BUILD_NUMBER}"
        SONAR_URL = "http://sonarqube:9000"
        GIT_REPO_NAME = "jenkins-sonarqube-argocd-project"
        GIT_USER_NAME = "abelshanoj"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/abelshanoj/jenkins-sonarqube-argocd-project.git'
            }
        }

        stage('Build and Test') {
            steps {
                sh 'cd app && mvn clean package'
            }
        }

        stage('Static Code Analysis') {
            environment {
                SONAR_AUTH_TOKEN = credentials('sonarqube')
            }
            steps {
                sh "cd app && mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2184:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}"
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                sh 'apt-get update && apt-get install -y docker.io'
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${DOCKER_IMAGE} -f app/Dockerfile app"
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Update Deployment File') {
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config user.email "jenkins@ci.local"
                        git config user.name "jenkins-ci"
                        sed -i "s|abelshanoj/spring-boot-app:.*|${DOCKER_IMAGE}|g" manifests/deployment.yml
                        git add manifests/deployment.yml
                        git commit -m "Update image to ${BUILD_NUMBER} [ci skip]"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main
                    '''
                }
            }
        }
    }
}
