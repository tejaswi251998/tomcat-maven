pipeline {
    agent { label 'agent' }

    environment {
        TOMCAT_HOME = "/opt/tomcat10"
        DEPLOY_SERVER = "ubuntu@44.193.0.46"
        APP_NAME = "myapp"
        SEVERITY_THRESHOLD = "HIGH"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh '''
                    mvn clean test
                '''
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package WAR') {
            steps {
                sh '''
                    mvn clean package -DskipTests
                    ls -lh target/*.war
                '''
            }
        }

        stage('Security Scan - Trivy') {
            steps {
                sh '''
                    trivy fs --exit-code 0 --format json --output trivy-report.json .

                    HIGH_COUNT=$(trivy fs --severity HIGH --exit-code 0 . | grep -c HIGH || true)

                    if [ "$HIGH_COUNT" -gt 0 ]; then
                        echo "❌ HIGH vulnerabilities found"
                        exit 1
                    fi
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }

        stage('Deploy to Tomcat App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        WAR_FILE=$(ls target/*.war)

                        echo "Deploying $WAR_FILE to Tomcat..."

                        ssh -o StrictHostKeyChecking=no $DEPLOY_SERVER << EOF
                            rm -rf $TOMCAT_HOME/webapps/$APP_NAME
                            rm -f  $TOMCAT_HOME/webapps/$APP_NAME.war
                        EOF

                        scp -o StrictHostKeyChecking=no $WAR_FILE \
                            $DEPLOY_SERVER:$TOMCAT_HOME/webapps/$APP_NAME.war

                        ssh -o StrictHostKeyChecking=no $DEPLOY_SERVER << EOF
                            $TOMCAT_HOME/bin/shutdown.sh || true
                            sleep 5
                            $TOMCAT_HOME/bin/startup.sh
                        EOF

                        echo "✅ Tomcat Deployment Completed"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
