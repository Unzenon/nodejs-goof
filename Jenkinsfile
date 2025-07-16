pipeline {
    agent none
    environment {
        DOCKERHUB_CREDENTIALS = credentials('DockerLogin')
        SNYK_CREDENTIALS = credentials('SnykToken')
        SONARQUBE_CREDENTIALS = credentials('SonarToken')
    }
    stages {
        stage('Secret Scanning Using Trufflehog'){
            agent {
                docker {
                    image 'trufflesecurity/trufflehog:latest'
                    args '--entrypoint='
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'trufflehog filesystem . --exclude-paths trufflehog-excluded-paths.txt --fail --json --no-update > trufflehog-scan-result.json'
                }
                sh 'cat trufflehog-scan-result.json'
                archiveArtifacts artifacts: 'trufflehog-scan-result.json'
            }
        }
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
        stage('SCA Snyk Test') {
            agent {
                docker {
                    image 'snyk/snyk:node'
                    args '--user root --network host --env SNYK_TOKEN=$SNYK_CREDENTIALS_PSW --entrypoint='
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'snyk test --json > snyk-scan-report.json'
                }
                sh 'cat snyk-scan-report.json'
                archiveArtifacts artifacts: 'snyk-scan-report.json'
            }
        }
        stage('SCA Retire Js') {
            agent {
                docker {
                    image 'node:lts-buster-slim'
                }
            }
            steps {
                sh 'npm install retire'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './node_modules/retire/lib/cli.js --outputformat json --outputpath retire-scan-report.json'
                }
                sh 'cat retire-scan-report.json'
                archiveArtifacts artifacts: 'retire-scan-report.json'
            }
        }
        // stage('SCA OWASP Dependency Check') {
        //     agent {
        //         docker {
        //             image 'owasp/dependency-check:latest'
        //             args '-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint='
        //         }
        //     }
        //     steps {
        //         catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
        //             sh '/usr/share/dependency-check/bin/dependency-check.sh --scan . --project "NodeJS Goof" --format ALL --noupdate'
        //         }
        //         archiveArtifacts artifacts: 'dependency-check-report.html'
        //         archiveArtifacts artifacts: 'dependency-check-report.json'
        //         archiveArtifacts artifacts: 'dependency-check-report.xml'
        //     }
        // }
        stage('SCA Trivy Scan Dockerfile Misconfiguration') {
            agent {
                docker {
                    image 'aquasec/trivy:latest'
                    args '-u root --network host --entrypoint='
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh 'trivy config Dockerfile --exit-code=1 --format json > trivy-scan-dockerfile-report.json'
            }
            sh 'cat trivy-scan-dockerfile-report.json'
            archiveArtifacts artifacts: 'trivy-scan-dockerfile-report.json'
            }
        }
        stage('SAST Snyk') {
            agent {
                docker {
                    image 'snyk/snyk:node'
                    args '-u root --network host'
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'snyk code test --json > snyk-sast-report.json'
                }
                sh 'cat snyk-sast-report.json'
                archiveArtifacts artifacts: 'snyk-sast-report.json'
            }
        }
        stage('SAST SonarQube') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest'
                    args '--network host -v .:/usr/src'
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=nodejs-goof \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://192.168.0.78:9000 \
                        -Dsonar.login=$SONARQUBE_TOKEN \
                        -Dsonar.qualitygate.wait=true
                    '''
                }
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
                sh 'docker build -t habibana028/nodejsgoof:0.1 .'
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker push habibana028/nodejsgoof:0.1'
            }
        }
        stage('Deploy Docker Image') {
            agent {
                docker {
                    image 'kroniak/ssh-client'
                    args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'DeploymentSSHKey', keyFileVariable: 'keyfile')]) {
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deploy@192.168.0.78 "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deploy@192.168.0.78 docker pull habibana028/nodejsgoof:0.1'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deploy@192.168.0.78 docker rm --force mongodb || true'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deploy@192.168.0.78 docker run --detach --name mongodb -p 27017:27017 mongo:3'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deploy@192.168.0.78 docker rm --force nodejsgoof || true'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deploy@192.168.0.78 docker run -it --detach --name nodejsgoof --network host habibana028/nodejsgoof:0.1'
                }
            }
        }
    }
}
