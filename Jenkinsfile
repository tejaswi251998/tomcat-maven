pipeline {
    agent { label 'maven-agent' }

    environment {
        APP_NAME    = "springboot-app"
        APP_DIR     = "/opt/${APP_NAME}"
        DEPLOY_USER = "ubuntu"
        DEPLOY_HOST = "44.193.0.46"
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
        stage('Security Scan - OWASP Dependency Check') {
              environment {
                NVD_API_KEY = credentials('owasp-apikey')
                DC_DATA_DIR = "${WORKSPACE}/.dc-data"
                DC_OUT_DIR  = "${WORKSPACE}/dependency-check-report"
              }
              steps {
                sh '''
                  set -e
                  rm -rf "$DC_DATA_DIR" "$DC_OUT_DIR"
                  mkdir -p "$DC_DATA_DIR" "$DC_OUT_DIR"
                '''
        
                dependencyCheck(
                  odcInstallation: 'dependency-check',
                  additionalArguments: """
                    --scan .
                    --format HTML
                    --out "${DC_OUT_DIR}"
                    --data "${DC_DATA_DIR}"
                    --nvdApiKey "${NVD_API_KEY}"
                    --failOnCVSS 7
                    --disableAssembly
                  """
                )
              }
              post {
                always {
                  archiveArtifacts artifacts: 'dependency-check-report/**', fingerprint: true, allowEmptyArchive: true
                }
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
                branch 'main'
            }
            steps {
                sshagent(['app-server']) {
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@44.193.0.46 "mkdir -p /opt/${APP_NAME}"'

                    sh 'scp -o StrictHostKeyChecking=no target/*.jar ubuntu@44.193.0.46:/opt/${APP_NAME}/app.jar'
                    
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@44.193.0.46 "pkill -f app.jar" || true'
                    
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@44.193.0.46 "nohup java -jar /opt/springboot-app/app.jar > /opt/${APP_NAME}/app.log 2>&1 &"'


                }
            }
        }



    }

    post {
        success {
            echo "🎉 SUCCESS on branch ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ FAILED on branch ${env.BRANCH_NAME}"
        }
    }
} 
