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
        WORKSPACE = "/home/vagrant/vagrant_folder/workspace/app-multibranched_changes_Samuel@2"
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
                    "-Dsonar.sources=. -Dsonar.javascrip.lcov.reportPaths=$WORKSPACE/coverage/lcov.info"
                    withSonarQubeEnv('sonarqube-automation') {
                        sh "${scannerHome}/bin/sonar-scanner ${scannerParameters}"
                    }
                }
            }
        }

        stage('Quality gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
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

        stage('nexus login') {
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

        //Continuous Deployment pipeline

        stage('Nexus login') {
            environment {
                NEXUS_CREDS = credentials("Nexus-creds")
            }
            steps {
                sh "sudo docker login -u $NEXUS_CREDS_USR -p $NEXUS_CREDS_PSW $NEXUS_SERVER_URL"
            }
        }

        stage('Deploy stage') {
            environment {
                NEXUS_REPO_IMAGE = "$NEXUS_SERVER_URL/$IMAGE_NAME"
            }
            steps {
                sh "sudo docker run -d --name test$BUILD_NUMBER -p 3000:3000 -v /home/vagrant/vagrant_folder/workspace/keys:/app/keys --network stack_atnet $NEXUS_REPO_IMAGE"
            }
        }

        stage('User acceptance test') {
            environment {
                API_BASE_URL = "http://10.0.2.15"
                PORT = "3000"
                SCENARIO_OPTION = "scenario/123456789"
                IMAGE_NAME = "msmapp$BUILD_NUMBER"
                NEXUS_SERVER_URL = "10.0.2.15:8082"
                NEXUS_REPO_IMAGE = "$NEXUS_SERVER_URL/$IMAGE_NAME"
            }
            steps {
                sh "curl -I $API_BASE_URL:$PORT/$SCENARIO_OPTION --silent | grep 200"
            }
        }

        stage('Clean VM') {
            environment {
                API_BASE_URL = "http://10.0.2.15"
                PORT = "3000"
                SCENARIO_OPTION = "scenario/123456789"
                IMAGE_NAME = "msmapp$BUILD_NUMBER"
                NEXUS_SERVER_URL = "10.0.2.15:8082"
                NEXUS_REPO_IMAGE = "$NEXUS_SERVER_URL/$IMAGE_NAME"
            }
            steps {
                sh "sudo docker stop test$BUILD_NUMBER"
                sh "sudo docker rm test$BUILD_NUMBER"
                sh "sudo docker rmi -f $NEXUS_REPO_IMAGE"
                sh "sudo docker logout $NEXUS_SERVER_URL"
            }
        }
    }
}