pipeline {
    agent { label 'test-node' }

    environment {
        APP_NAME    = "springboot-app"
        APP_DIR     = "/opt/${APP_NAME}"
        DEPLOY_USER = "ubuntu"
        DEPLOY_HOST = "34.239.1.185"
        JAVA_HOME   = "/usr/lib/jvm/java-21-openjdk-amd64"
        PATH        = "${JAVA_HOME}/bin:${env.PATH}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                      mvn sonar:sonar \
                      -Dsonar.projectKey=petclinic \
                      -Dsonar.projectName=petclinic \
                      -Dsonar.host.url=http://<SONAR_HOST>:9000 \
                      -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Quality Gate Info') {
            steps {
                echo "Quality Gate status must be reviewed manually in SonarQube UI (Free Edition limitation)."
            }
        }


        stage('Package JAR') {
            steps {
                sh '''
                    mvn clean package -DskipTests
                    ls -lh target/*.jar
                '''
            }
        }

        stage('Security Scan (Trivy)') {
            steps {
                sh '''
                    trivy fs --exit-code 0 --format json \
                    --output trivy-report.json .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }
        stage('Deploy to App EC2 (main only)') {
            when {
                branch 'develop'
            }
            steps {
                sshagent(['app-server']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${DEPLOY_HOST} "mkdir -p /opt/springboot-app"
                        scp -o StrictHostKeyChecking=no target/*.jar ubuntu@${DEPLOY_HOST}:/opt/springboot-app/app.jar
                        ssh -o StrictHostKeyChecking=no ubuntu@${DEPLOY_HOST} "pkill -f app.jar" || true
                        ssh -o StrictHostKeyChecking=no ubuntu@${DEPLOY_HOST} "nohup java -jar /opt/springboot-app/app.jar > /opt/springboot-app/app.log 2>&1 &"
                        """

                }
            }
        }



    }

    post {
        success {
            echo "🎉 SUCCESS on branch ${env.BRANCH_NAME}"
            slackSend(
                message: "✅ SUCCESS: Job ${env.JOB_NAME} #${env.BUILD_NUMBER} on branch ${env.BRANCH_NAME}"
            )
                emailext(
                    subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: "Build succeeded on branch ${env.BRANCH_NAME}",
                    to: "tejaswi98e@gmail.com"
                )
            
        }
        failure {
            echo "❌ FAILED on branch ${env.BRANCH_NAME}"
        }
    }
} 
