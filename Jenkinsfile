pipeline { 
    agent none 

    environment { 
        DOCKERHUB_CREDENTIALS = credentials('DockerLogin') 
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
                withCredentials([usernamePassword(credentialsId: 'DockerLogin', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'docker build -t habibana028/nodejs-goof:0.1 .'
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker push habibana028/nodejs-goof:0.1'
                }
            }
        }

        stage('Deploy Docker Image') { 
            agent {
                docker {
                    image 'kroniak/ssh-client'
                    args '--user root'
                }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: "DeploymentSSHKey", keyFileVariable: 'keyfile'),
                                 usernamePassword(credentialsId: 'DockerLogin', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    
                    sh '''
                        ssh -i ${keyfile} -o StrictHostKeyChecking=no ubuntu@192.168.100.43 "
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin &&
                        docker pull habibana028/nodejs-goof:0.1 &&
                        docker rm -f mongodb || true &&
                        docker run -d --name mongodb -p 27017:27017 mongo:3 &&
                        docker rm -f nodejs || true &&
                        docker run -it -d -p 3001:3001 --name nodejs --network host habibana028/nodejs-goof:0.1
                        "
                    '''
                }
            }
        }   
    }
}
