pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Extract Commit SHA') {
            steps {
                script {
                    env.COMMIT_SHA = sh(script: "git rev-parse --short=7 HEAD", returnStdout: true).trim()
                    echo "Commit: ${env.COMMIT_SHA}"
                }
            }
        }

        stage('Install & Test') {
            steps {
                script {
                    // הורדה והפעלה מקומית של Node.js בתוך ה-Workspace (ללא פגיעה באבטחת המערכת או צורך ב-Root)
                    sh '''
                        export NODE_VERSION=18.16.0
                        export PATH=$WORKSPACE/node-v$NODE_VERSION-linux-x64/bin:$PATH
                        
                        if [ ! -d "node-v$NODE_VERSION-linux-x64" ]; then
                            echo "Downloading Node.js locally..."
                            curl -O https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.gz
                            tar -xzf node-v$NODE_VERSION-linux-x64.tar.gz
                        fi
                        
                        npm install
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_NAME = "wubdigitalinfo/holiday-app"
                    sh "docker build -t ${env.IMAGE_NAME}:${env.BUILD_NUMBER} -t ${env.IMAGE_NAME}:latest ."
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh 'docker push $IMAGE_NAME:$BUILD_NUMBER'
                    sh 'docker push $IMAGE_NAME:latest'
            }
        }

        stage('Deploy with Ansible') {
            when {
                expression { env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master' }
            }
            steps {
                // הרצת ה-Playbook של Ansible לעדכון אוטומטי של השרת
                sh "ansible-playbook -i ansible/inventory ansible/deploy.yml --extra-vars 'image_tag=${env.BUILD_NUMBER}'"
            }
        }

                        stage('Health Check') {
            when {
                expression { env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master' }
            }
            steps {
                script {
                    def response = sh(
                        script: "ssh cs.humble-chainsaw-4j9rjw79575wf7pp5.main curl -s http://localhost:3000/health",
                        returnStdout: true
                    ).trim()
                    echo "Health check response: ${response}"

                    if (!response.contains('"status":"healthy"')) {
                        error("Health check failed! App is not healthy on target server.")
                    }
                    echo "Health check passed - application is running correctly."
                }
            }
        }
    }

    post {
        success { 
            echo "Pipeline succeeded on branch ${env.BRANCH_NAME}" 
        }
        failure { 
            echo "Pipeline failed on branch ${env.BRANCH_NAME}" 
        }
    }
}