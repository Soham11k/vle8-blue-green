```groovy
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

                sh '''
                    sudo -u ubuntu /usr/local/bin/minikube image load myapp:v2
                '''
            }
        }

        stage('Deploy Green') {
            steps {
                sh '''
                    sudo -u ubuntu /snap/bin/kubectl apply -f deployment-green.yaml

                    sudo -u ubuntu /snap/bin/kubectl \
                        rollout status deployment/myapp-green \
                        --timeout=180s
                '''
            }
        }

        stage('Ensure Service Exists') {
            steps {
                sh '''
                    sudo -u ubuntu /snap/bin/kubectl apply -f service.yaml
                '''
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

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    set -eu

                    echo "Checking Docker..."
                    docker info >/dev/null

                    echo "Preparing ZAP report directory..."

                    ZAP_DIR="/tmp/zap-report-${BUILD_NUMBER}"
                    mkdir -p "$ZAP_DIR"
                    chmod 777 "$ZAP_DIR"

                    mkdir -p "$WORKSPACE/zap-report"

                    echo "Starting Kubernetes port-forward..."

                    nohup sudo -u ubuntu /snap/bin/kubectl \
                        port-forward \
                        -n default \
                        service/myapp-service \
                        18089:80 \
                        --address 127.0.0.1 \
                        >/tmp/zap-port-forward.log 2>&1 &

                    PF_PID=$!

                    cleanup() {
                        kill "$PF_PID" 2>/dev/null || true
                    }

                    trap cleanup EXIT

                    echo "Waiting for application..."

                    READY=0

                    for i in $(seq 1 30); do
                        if curl -fsS \
                            http://127.0.0.1:18089/ \
                            >/dev/null; then
                            READY=1
                            break
                        fi

                        sleep 2
                    done

                    if [ "$READY" -ne 1 ]; then
                        cat /tmp/zap-port-forward.log
                        echo "ERROR: Application is not reachable."
                        exit 1
                    fi

                    echo "Application is reachable."
                    echo "Running OWASP ZAP baseline scan..."

                    docker run --rm \
                        --user 0:0 \
                        --network host \
                        -v "$ZAP_DIR:/zap/wrk/:rw" \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://127.0.0.1:18089 \
                        -m 1 \
                        -I \
                        -r /zap/wrk/zap-report.html

                    echo "Checking generated report..."

                    if [ ! -s "$ZAP_DIR/zap-report.html" ]; then
                        echo "ERROR: ZAP HTML report was not generated."
                        exit 1
                    fi

                    cp "$ZAP_DIR/zap-report.html" \
                       "$WORKSPACE/zap-report/zap-report.html"

                    echo "ZAP HTML report copied successfully."
                '''
            }

            post {
                always {
                    archiveArtifacts(
                        artifacts: 'zap-report/zap-report.html',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    echo "Checking Kubernetes deployments..."

                    sudo -u ubuntu /snap/bin/kubectl get deployments

                    echo "Checking service endpoints..."

                    sudo -u ubuntu /snap/bin/kubectl get endpoints myapp-service

                    echo "Checking application response..."

                    curl -fsS --max-time 10 \
                        http://127.0.0.1:18089/ \
                        >/dev/null || true
                '''
            }
        }
    }
}
```
