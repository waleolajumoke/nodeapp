pipeline{
    agent any
    environment{
        DOCKERHUB_USERNAME = "tech365"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/nodejs-cicd-app"
        APP_SERVER = "ec2-3-94-80-117.compute-1.amazonaws.com"
        APP_PORT = "3000"
        APP_USER = "ec2-user"
        CONTAINER_NAME = "nodejs-app"
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

        // stage("Run tests"){
        //     steps{
        //         sh "npm test  --if-present"
        //     }
        // }
        
        stage("Build docker image"){
            steps{
                sh '''
                docker build \
                -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                -t ${IMAGE_NAME}:latest .
                '''
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
                    docker push ${IMAGE_NAME}:latest
                    docker logout
                    '''
                }
            }
        }

        stage("Deploy to AWS ec2"){
            steps{
                sshagent(credentials: ['ec2-ssh-key']){
                    sh '''
                    mkdir -p ~/.ssh
                    chmod 700 ~/.ssh
                    ssh-keyscan -H "${APP_SERVER}" >> ~/.ssh/known_hosts
                    chmod 600 ~/.ssh/known_hosts

                    ssh "${APP_USER}@${APP_SERVER}" "
                    set -e
                    docker pull '${IMAGE_NAME}:latest'
                    docker stop '${CONTAINER_NAME}' || true
                    docker rm '${CONTAINER_NAME}' || true

                    docker run -d --name '${CONTAINER_NAME}' --restart unless-stopped -p '${APP_PORT}:3000' '${IMAGE_NAME}:latest'
                    "
                    '''
                }
            }
        }
        stage('verify deployment'){
            steps{
                sh ''' 
                sleep 10
                curl --fail --retry 5 --retry-delay 5 "http://${APP_SERVER}:${APP_PORT}"
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