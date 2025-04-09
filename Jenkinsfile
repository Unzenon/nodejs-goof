pipeline { 
    agent none 
    environment { 
        DOCKERHUB_CREDENTIALS credentials('DockerLogin') 
    } 
    stages { 
        stage('Build') {
            agent {
                docker {
                    image 'node:lts-buster-slim'
                }
            }
            steps {
                sh 'npm install'
            }
        } 
        stage('Build Docker Image and Push to Docker Registry') { 
            agent {
                docker {
                    image 'docker:dind'
                    args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'docker build -t habibana028/nodejs-goof:0.1 .'
                sh 'echo $DOCKERHUB_CREDENTIALS-PSW | docker login -u $DOCKERHUB_CREDENTIALS-ID --password-stdin'
                sh 'docker push habibana028/nodejs-goof:0.1'
            }
        stage('Deploy Docker Image') { 
            agent {
                docker {
                    image 'kroniak/ssh-client'
                    args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: "DeploymentSSHKey", keyFileVariable: 'keyfile')]) {
                    sh 'ssh -i ${keyfile} -o StrickHostKeyChecking=no ubuntu@192.168.100.43 "echo $DOCKERHUB_CREDENTIALS-PSW | docker login -u $DOCKERHUB_CREDENTIALS-ID --password-stdin DOCKERHUB_CREDENTIALS-PSW | docker login -u $DOCKERHUB_CREDENTIALS-ID --password-stdin "'
                    sh 'ssh -i ${keyfile} -o StrickHostKeyChecking=no ubuntu@192.168.100.43 docker pull habibana028/nodejs-goof:0.1'
                    sh 'ssh -i ${keyfile} -o StrickHostKeyChecking=no ubuntu@192.168.100.43 docker rm -f mongodb'
                    sh 'ssh -i ${keyfile} -o StrickHostKeyChecking=no ubuntu@192.168.100.43 run -d --name mongodb -p 27017:27017 mongo:3'
                    sh 'ssh -i ${keyfile} -o StrickHostKeyChecking=no ubuntu@192.168.100.43 rm -f nodejs-goof'
                    sh 'ssh -i ${keyfile} -o StrickHostKeyChecking=no ubuntu@192.168.100.43 run -it -d -p 3001:3001 --name nodejs --network host habibana028/nodejs-goof:0.1'
                }
            }
            }
        }   
    }
}
