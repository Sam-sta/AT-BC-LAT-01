pipeline {
    agent {label 'linux'}
    environment {
        IMAGE_NAME = "msmapp$BUILD_NUMBER"
        TARGET_IMAGE = "samsta/practice_jenkins:$IMAGE_NAME"
        DOCKERHUB_CREDS = credentials('Dockerhub-login')
        PROJECT_NAME = "msm-node-app"
        NEXUS_SERVER_URL = "10.0.2.15:8082"
        PRIVATE_IMAGE_NAME = "$NEXUS_SERVER_URL/$IMAGE_NAME"
        NEXUS_CREDS = credentials("Nexus-creds")
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

        stage('static code analysis with sonarqube') {
            steps {
                script {
                    def scannerHome = tool 'SonarQube Scanner'
                    def scannerParameters = "-Dsonar.projectName=$PROJECT_NAME -Dsonar.projectKey=$PROJECT_NAME " +
                    "-Dsonar.sources=. -Dsonar.javascrip.lcov.reportPaths=coverage/lcov.info"
                    withSonarQubeEnv('sonarqube-automation') {
                        sh "${scannerHome}/bin/sonar-scanner ${scannerParameters}"
                    }
                }
            }
        }

        stage('build app image') {
            // when {
            //     branch "main"
            // }
            steps {
                sh "sudo docker build -t $PRIVATE_IMAGE_NAME ."
            }
            post {
                failure {
                    script {
                        sh "sudo docker rmi \$(docker images --filter dangling=true -q)"
                    }
                }
            }
        }

        stage('docker-hub login') {
            // when {
            //     branch "main"
            // }
            steps {
                sh "sudo docker login -u $NEXUS_CREDS_USR -p $NEXUS_CREDS_PSW $NEXUS_SERVER_URL"
            }
        }

        stage('push image') {
            // when {
            //     branch "main"
            // }
            steps {
                sh "sudo docker push $PRIVATE_IMAGE_NAME"
            }
            post {
                always {
                    script {
                        sh "sudo docker rmi -f $PRIVATE_IMAGE_NAME"
                        sh "sudo docker logout $NEXUS_SERVER_URL"
                    }
                }
            }
        }
    }
}