pipeline {
    agent any
    triggers {
        githubPush()
    }
    environment {
        REGISTRY = 'vigneshsiva94'
        IMAGE_NAME = 'bankapp'
        KUBE_CONFIG = credentials('kubeconfig')
        SONARQUBE_CREDENTIALS = credentials('sonarqube-token')
        SONARQUBE_URL = 'http://localhost:9000'
        NEXUSURL ='http://localhost:8081'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/vignesh-awsdevops/Multi-Tier-with-Database.git'
            }
        }
        
        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Maven Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar -Dsonar.host.url=${SONARQUBE_URL} -Dsonar.login=${SONARQUBE_CREDENTIALS}'
                }
            }
        }
        
        stage('Security Scan - Trivy') {
            steps {
                sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL . || true'
            }
        }
        
        stage('Security Scan - OWASP Dependency-Check') {
            steps {
                sh 'mvn org.owasp:dependency-check-maven:check'
            }
        }
        
        stage('Publish to Nexus') {
            steps {
                sh "mvn deploy -DaltDeploymentRepository=nexus::default::${NEXUS_URL}/repository/maven-releases/ -Dnexus.username=${NEXUS_CREDENTIALS_USR} -Dnexus.password=${NEXUS_CREDENTIALS_PSW}"
            }
        }        
        stage('Docker Build & Push') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'docker-credentials') {
                        sh "docker build -t ${REGISTRY}/${IMAGE_NAME}:latest ."
                        sh "docker push ${REGISTRY}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }
        
        stage('Kubernetes Deployment') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh 'kubectl apply -f k8s/deployment.yaml'
                }
            }
        }
    }
    
    post {
        always {
            junit '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }
        failure {
            mail to: 'vigneshsiva007@gmail.com',
                 subject: 'Jenkins Build Failed: ${JOB_NAME} #${BUILD_NUMBER}',
                 body: "Check Jenkins for details: ${BUILD_URL}"
        }
        success {
            mail to: 'vigneshsiva007@gmail.com',
                 subject: 'Jenkins Build Successful: ${JOB_NAME} #${BUILD_NUMBER}',
                 body: "The build has completed successfully: ${BUILD_URL}"
        }
    }
}
