pipeline {
    agent {label 'linux'}
    environment {
        // NEXUS_SERVER_URL = "10.0.2.15:8082"
        // PRIVATE_IMAGE_NAME = "$NEXUS_SERVER_URL/$IMAGE_NAME"
        // NEXUS_CREDS = credentials("Nexus-creds")
        IMAGE_NAME = "msmapp$BUILD_NUMBER"
        TARGET_IMAGE = "samsta/practice_jenkins:$IMAGE_NAME"
        DOCKERHUB_CREDS = credentials('Dockerhub-login')
        PROJECT_NAME = "msm-node-app"
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
                sh "sudo docker build -t $IMAGE_NAME ."
            }
            post {
                failure {
                    script {
                        sh "sudo docker rmi \$(docker images --filter dangling=true -q)"
                    }
                }
            }
        }

        stage('dockerhub login') {
            // when {
            //     branch "main"
            // }
            steps {
                sh "sudo docker login -u $DOCKERHUB_CREDS_USR -p $DOCKERHUB_CREDS_PSW"
            }
        }

        stage('push image') {
            // when {
            //     branch "main"
            // }
            steps {
                sh "sudo docker tag $IMAGE_NAME $TARGET_IMAGE"
                sh "sudo docker push $TARGET_IMAGE"
            }
            post {
                always {
                    script {
                        sh "sudo docker rmi -f $TARGET_IMAGE"
                        sh "sudo docker rmi -f $IMAGE_NAME"
                        sh "sudo docker logout"
                    }
                }
            }
        }

        //Continuous Delivery pipeline

        stage('Continuous Delivery') {
            environment {
                API_BASE_URL = "http://10.0.2.15"
                PORT = "3000"
                SCENARIO_OPTION = "scenario/123456789"
                IMAGE_NAME = "msmapp$BUILD_NUMBER"
                LOCAL_IMAGE = "samsta/practice_jenkins:$IMAGE_NAME"
                TARGET_IMAGE = "samsta/practice_jenkins:$IMAGE_NAME-prod"
                DOCKERHUB_CREDS = credentials('Dockerhub-login')
                // NEXUS_SERVER_URL = "10.0.2.15:8082"
                // NEXUS_REPO_IMAGE = "$NEXUS_SERVER_URL/$IMAGE_NAME"
            }
            stages{
                stage('Dockerhub-login') {
                    steps {
                        sh "sudo docker login -u $DOCKERHUB_CREDS_USR -p $DOCKERHUB_CREDS_PSW"
                    }
                }

                stage('Deploy stage') {
                    steps {
                        sh "sudo docker run -d --name test$BUILD_NUMBER -p 3000:3000 -v /home/vagrant/vagrant_folder/workspace/keys:/app/keys --network stack_atnet $LOCAL_IMAGE"
                    }
                }

                // stage('User acceptance test') {
                //     steps {
                //         sh "curl -I $API_BASE_URL:$PORT/$SCENARIO_OPTION --silent | grep 200"
                //     }
                // }

                stage('Production tag') {
                    environment {
                        TAG = "$BUILD_NUMBER"
                    }
                    steps {
                        sh "sudo docker tag $LOCAL_IMAGE $TARGET_IMAGE"
                    }
                }

                stage('Deliver image to production') {
                    steps {
                        sh """
                        echo '$DOCKERHUB_CREDS_PSW' | sudo docker login -u $DOCKERHUB_CREDS_USR --password-stdin
                        sudo docker push $TARGET_IMAGE
                        """
                    }
                }

                stage('Clean VM') {
                    steps {
                        sh "sudo docker stop test$BUILD_NUMBER"
                        sh "sudo docker rm test$BUILD_NUMBER"
                        sh "sudo docker rmi -f $TARGET_IMAGE"
                        sh "sudo docker logout"
                    }
                }
            }
        }

        //Continuous deployment pipeline

        stage('Coninuous deployment') {
            environment {
                PROD_SERVER = "ubuntu@ec2-54-174-33-135.compute-1.amazonaws.com"
                TARGET_IMAGE = "samsta/practice_jenkins:$IMAGE_NAME-prod"
                DB_KEY = "/home/vagrant/vagrant_file/workspace/keys"
            }
            stages {
                stage('copy key to server') {
                    steps {
                        sshagent(['aws-key']) {
                            sh "ssh -o 'StrictHostKeyChecking no' $PROD_SERVER mkdir -p keys"
                            sh "scp $DB_KEY $PROD_SERVER:/home/ubuntu/keys"
                            sh "ssh -o 'StrictHostKeyChecking no' $PROD_SERVER ls /home/ubuntu/keys"
                        }
                    }
                }

                stage('deploy in production') {
                    steps {
                        sshagent(['aws-key']) {
                            sh "ssh -o 'StrictHostKeyChecking no' $PROD_SERVER sudo docker rm -f test"
                            sh "ssh -o 'StrictHostKeyChecking no' $PORD_SERVER echo '$DOCKERHUB_CREDS_PSW' | sudo docker login -$DOCKERHUB_CRES_USR --password-stdin"
                            sh "ssh -o 'StrictHostKeyChecking no' $PROD_SERVER sudo docker pull $TARGET_IMAGE"
                            sh "ssh -o 'StrictHostKeyChecking no' $PROD_SERVER sudo docker run -d --name test -p 3000:3000 -v /home/ubuntu/keys:/app/keys $TARGET_IMAGE"
                            sh "ssh -o 'StrictHostKeyChecking no' $PROD_SERVER sudo docker logout"
                        }
                    }
                }
            }
        }
    }
}