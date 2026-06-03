@Library("Shared") _
pipeline{
    agent { label "Pratik"}
    
    stages{
        
        stage("Hello"){
            steps{
                script{
                    hello()
                }
            }
        }
        
        stage("Code"){
            steps{
                script{
                    clone("https://github.com/pratikkamble14/django-notes-app.git","feature")
                }
            }
        }
        stage("Build"){
            steps{
                script{
                    docker_build("notes-app","latest","pratik140404")
                }
              
            }
            
        }
        stage("Push"){
            steps{
                script{
                    docker_push("notes-app","latest","pratik140404")
                }
            }
        }
        stage("Deploy"){
            steps{
                script{
                    docker_compose()
                }
                echo"Deployment Completed."
            }
            
        }
    }
}
