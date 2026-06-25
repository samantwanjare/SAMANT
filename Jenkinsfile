pipeline {

    agent {
        label 'samant'
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = '769265543964.dkr.ecr.us-east-1.amazonaws.com/samant/notes-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/samantwanjare/SAMANT.git'
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t notes-app:${BUILD_NUMBER} .'
            }
        }

        stage('Login To ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 769265543964.dkr.ecr.ap-south-1.amazonaws.com
                '''
            }
        }

        stage('Tag Image') {
            steps {
                sh '''
                docker tag notes-app:${BUILD_NUMBER} 769265543964.dkr.ecr.ap-south-1.amazonaws.com/samant/notes-app:${BUILD_NUMBER}
                docker tag notes-app:${BUILD_NUMBER} 769265543964.dkr.ecr.ap-south-1.amazonaws.com/samant/notes-app:latest
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                docker push 769265543964.dkr.ecr.ap-south-1.amazonaws.com/samant/notes-app:${BUILD_NUMBER}
                docker push 769265543964.dkr.ecr.ap-south-1.amazonaws.com/samant/notes-app:latest
                '''
            }
        }
    }
}
