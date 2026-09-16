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
                // הרצת npm install בתוך קונטיינר Node מבודד מבלי להתקין כלים על Jenkins
                sh 'docker run --rm -v ${WORKSPACE}:/app -w /app node:18 npm install'
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
                    sh "docker push ${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker push ${env.IMAGE_NAME}:latest"
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