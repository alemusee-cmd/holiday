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
        }

        stage('Deploy with Ansible') {
            when {
                expression { env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master' }
            }
            steps {
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

                    if (response.contains('"status":"healthy"')) {
                        sh "ssh cs.humble-chainsaw-4j9rjw79575wf7pp5.main 'echo ${env.BUILD_NUMBER} > /tmp/last_good_build.txt'"
                        echo "Health check passed - build ${env.BUILD_NUMBER} recorded as last known good version."
                    } else {
                        echo "Health check FAILED! Attempting automatic rollback..."

                        def lastGoodBuild = sh(
                            script: "ssh cs.humble-chainsaw-4j9rjw79575wf7pp5.main 'cat /tmp/last_good_build.txt 2>/dev/null || echo none'",
                            returnStdout: true
                        ).trim()

                        if (lastGoodBuild != "none" && lastGoodBuild != "") {
                            echo "Rolling back to last known good build: ${lastGoodBuild}"
                            sh "ansible-playbook -i ansible/inventory ansible/deploy.yml --extra-vars 'image_tag=${lastGoodBuild}'"

                            def rollbackCheck = sh(
                                script: "ssh cs.humble-chainsaw-4j9rjw79575wf7pp5.main curl -s http://localhost:3000/health",
                                returnStdout: true
                            ).trim()
                            echo "Post-rollback health check: ${rollbackCheck}"

                            error("Health check failed for build ${env.BUILD_NUMBER}. Automatically rolled back to build ${lastGoodBuild}.")
                        } else {
                            error("Health check failed for build ${env.BUILD_NUMBER}. No previous good build found - manual intervention required!")
                        }
                    }
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