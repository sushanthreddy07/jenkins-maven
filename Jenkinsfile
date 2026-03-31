pipeline {
    agent { label 'slave' }

    environment {
        APP_SERVER_IP   = '172.31.1.156'
        APP_SERVER_USER = 'ubuntu'
        DEPLOY_DIR      = '/opt/application'
    }

    stages {

        stage('Checkout Code') {
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
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Security Scan - Trivy') {
            steps {
                sh '''
                    if ! command -v trivy &> /dev/null; then
                        sudo apt-get install -y wget apt-transport-https gnupg
                        wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                        echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
                        sudo apt-get update
                        sudo apt-get install -y trivy
                    fi
                '''
                sh 'trivy fs --exit-code 1 --severity CRITICAL .'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar',
                    fingerprint: true
            }
        }

        stage('Deploy to App Server') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ['app-server-ssh']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no \
                            target/*.jar \
                            ubuntu@''' + env.APP_SERVER_IP + ''':/opt/application/app.jar
                    '''
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@''' + env.APP_SERVER_IP + ''' \
                            "cd /opt/application && nohup java -jar app.jar > app.log 2>&1 &"
                    '''
                }
            }
        }
    }

 post {
        success {
            mail to: 'rachalasushanth007@gmail.com',
                 subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build succeeded.\nCheck: ${env.BUILD_URL}"
        }
        failure {
            mail to: 'rachalasushanth007@gmail.com',
                 subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build failed.\nCheck: ${env.BUILD_URL}"
        }
    }
