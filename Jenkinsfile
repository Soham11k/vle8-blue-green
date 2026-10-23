pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            '${scannerHome}/bin/sonar-scanner' \
                              -Dsonar.projectKey=vle8-blue-green \
                              -Dsonar.projectName='VLE8 Blue Green Deployment' \
                              -Dsonar.sources=. \
                              -Dsonar.exclusions=.git/**,**/*.yaml,**/*.yml,**/Dockerfile,**/Jenkinsfile
                        """
                    }
                }
            }
        }

        stage('Build Green') {
            steps {
                sh 'docker build -t myapp:v2 .'
                sh 'sudo -u ubuntu /usr/local/bin/minikube image load myapp:v2'
            }
        }

        stage('Deploy Green') {
            steps {
                sh 'sudo -u ubuntu /snap/bin/kubectl apply -f deployment-green.yaml'
                sh 'sudo -u ubuntu /snap/bin/kubectl rollout status deployment/myapp-green --timeout=180s'
            }
        }
        stage('Ensure Service Exists') {
            steps {
                sh 'sudo -u ubuntu /snap/bin/kubectl apply -f service.yaml'
            }
        }

        stage('Ensure Service Exists') {
            steps {
                sh 'sudo -u ubuntu /snap/bin/kubectl apply -f service.yaml'
            }
        }

        stage('Approve Switch') {
            steps {
                input message: 'Green is ready. Switch traffic?',
                      ok: 'Switch to Green'
            }
        }

        stage('Switch Traffic') {
            steps {
                sh '''
                sudo -u ubuntu /snap/bin/kubectl patch service myapp-service \
                  -p '{"spec":{"selector":{"app":"myapp","color":"green"}}}'
                '''
            }
        }

        stage('Verify') {
            steps {
                sh 'sudo -u ubuntu /snap/bin/kubectl get deployments'
                sh 'sudo -u ubuntu /snap/bin/kubectl get endpoints myapp-service'
            }
        }
    }
}
