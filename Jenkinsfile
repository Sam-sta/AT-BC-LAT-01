pipeline {
    agent {label 'linux'}
    environment {
        IMAGE_NAME = "msmapp:$BUILD_NUMBER"
        TARGET_IMAGE = "samsta/practice_jenkins:$IMAGE_NAME"
        DOCKERHUB_CREDS = credentials('Dockerhub-login')
    }
    tools {
        nodejs 'NodeJS 14.17.5'
    }
    stages {
        stage('install dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('test code') {
            steps {
                sh "npm test"
            }
        }

        stage('build app image') {
            // when {
            //     branch "main"
            // }
            steps {
                sh "sudo docker build -t $IMAGE_NAME ."
            }
        }

        stage('docker-hub login') {
            // when {
            //     branch "main"
            // }
            steps {
                sh "sudo docker login -u $DOCKERHUB_CREDS_USR -p $DOCKERHUB_CREDS_PSW"
            }
        }

        stage('tag image') {
            // when {
            //     branch "main"
            // }
            steps {
                sh "sudo docker tag $IMAGE_NAME $TARGET_IMAGE"
            }
        }

        stage('push image') {
            /*when {
                branch "main"
            }*/
            steps {
                sh "sudo docker push $TARGET_IMAGE"
            }
            post {
                always {
                    script {
                        sh "sudo docker rmi -f $TARGET_IMAGE"
                        sh "sudo docker rmi -f $TIMAGE_NAME"
                        sh "sudo docker logout"
                    }
                }
            }
        }
    }
}