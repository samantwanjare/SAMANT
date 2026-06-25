@Library("shared") _
pipeline {
agent {
    label "samant"
}

environment {
    ECR_REGISTRY = "769265543964.dkr.ecr.ap-south-1.amazonaws.com"
    ECR_REPOSITORY = "samant/notes-app"
    IMAGE_TAG = "latest"
    AWS_REGION = "ap-south-1"
}

stages {
    stage("Hello"){
        steps{
            script{
                hello()
            }
        }
    }
    stage("Code") {
        steps {
            script{
                clone("https://github.com/samantwanjare/SAMANT.git","main")
            }
   
        }
    }

    stage("Build") {
        steps {
            echo "Building a Docker image"

            sh '''
            whoami
            docker build -t notes-app:${IMAGE_TAG} .
            '''
        }
    }

    stage("Push to ECR") {
        steps {
            echo "Pushing Docker image to AWS ECR"

            sh '''
            aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin ${ECR_REGISTRY}

            docker tag notes-app:${IMAGE_TAG} \
            ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}

            docker push \
            ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
            '''
        }
    }

    stage("Deploy") {
        steps {
            echo "Deploying application"

            sh '''
            docker compose up -d
            '''
        }
    }
}

post {

    success {
        echo "Pipeline executed successfully"
    }

    failure {
        echo "Pipeline failed"
    }
  }
}
