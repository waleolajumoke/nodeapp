pipeline{
    agent any
    environment{
        DOCKERHUB_USERNAME = "tech365"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/nodejs-cicd-app"
        APP_SERVER = "3.94.80.117/"
        APP_PORT = "3000"
    }

    stages{
        stage("checkout"){
            steps{
                checkout scm
            }
        }

        stage("Install dependencies"){
            steps{
                sh "npm ci"
            }
        }

        stage("Run tests"){
            steps{
                sh "npm test"
            }
        }
        
        stage("Build docker image"){
            steps{
                sh docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .
            }
        }
        stage("Push to docker hub"){
            steps{
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]){
                    sh '''
                    echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USERNAME" --password-stdin
                    docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                    docker push $IMAGE_NAME}:latest
                    docker logout
                    '''
                }
            }
        }

        stage("Deploy to AWS ec2"){
            steps{
                sshagent(credentials: ['aws-credentials']){
                    sh '''
                    ssh -i "aghokey.pem" ec2-user@ec2-3-94-80-117.compute-1.amazonaws.com
                    docker pull ${IMAGE_NAME}:latest &&
                    docker stop nodejs-app || true &&
                    docker rm nodejs-app || true &&

                    docker run -d --name nodejs-app --restart unless stopped -p ${APP_PORT}:3000 ${IMAGE_NAME}:latest
                    '''
                }
            }
        }
        post{
            success{
                echo "application deployed successfully"
            }
            failure{
                echo "pipeline failed. check jenkins console output"
            }
            always{
                sh 'docker image prune -f || true'
            }
        }

}

}