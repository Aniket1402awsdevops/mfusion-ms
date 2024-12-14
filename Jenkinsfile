pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "879381286690"
        REGION = "ap-south-2"
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
        IMAGE_NAME = "Aniket1402awsdevops/mfusion-ms:mfusion-ms-v.1.${env.BUILD_NUMBER}"
        ECR_IMAGE_NAME = "${ECR_URL}/mfusion-ms:mfusion-ms-v.1.${env.BUILD_NUMBER}"
        KUBECONFIG_ID = 'kubeconfig-aws-aks-k8s-cluster'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    }

    tools {
        maven 'maven_3.9.4'
    }

    stages {
        stage('Build and Test for Dev') {
            when {
                branch 'dev'
            }
            stages {
                stage('Code Compilation') {
                    steps {
                        echo 'Code Compilation is In Progress!'
                        sh 'mvn clean compile'
                        echo 'Code Compilation is Completed Successfully!'
                    }
                }

                stage('Code QA Execution') {
                    steps {
                        echo 'JUnit Test Case Check in Progress!'
                        sh 'mvn clean test'
                        echo 'JUnit Test Case Check Completed!'
                    }
                }

                stage('Code Package') {
                    steps {
                        echo 'Creating WAR Artifact'
                        sh 'mvn clean package'
                        echo 'Artifact Creation Completed'
                    }
                }

                stage('Building & Tag Docker Image') {
                    steps {
                        echo "Starting Building Docker Image: ${env.IMAGE_NAME}"
                        sh "docker build -t ${env.IMAGE_NAME} ."
                        echo 'Docker Image Build Completed'
                    }
                }

                // Skipping Docker Hub Push Stage
                // stage('Docker Push to Docker Hub') {
                //     steps {
                //         withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB_CRED', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                //             echo "Pushing Docker Image to DockerHub: ${env.IMAGE_NAME}"
                //             sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin"
                //             sh "docker push docker.io/${env.IMAGE_NAME}"
                //             echo "Docker Image Push to DockerHub Completed"
                //         }
                //     }
                // }

                stage('Docker Image Push to Amazon ECR') {
                    steps {
                        echo "Tagging Docker Image for ECR: ${env.ECR_IMAGE_NAME}"
                        sh "docker tag ${env.IMAGE_NAME} ${env.ECR_IMAGE_NAME}"
                        echo "Docker Image Tagging Completed"

                            withDockerRegistry([credentialsId: 'aws-ecr-credentials', url: "https://${ECR_URL}"]) {
                            echo "Pushing Docker Image to ECR: ${env.ECR_IMAGE_NAME}"
                            sh "docker push ${env.ECR_IMAGE_NAME}"
                            echo "Docker Image Push to ECR Completed"
                        }
                    }
                }
            }
        }

        // Other stages for deployments
        stage('Deploy to Development Environment') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    echo "Deploying to Dev Environment"
                    def yamlFiles = ['00-ingress.yaml', '02-service.yaml', '03-service-account.yaml', '04-deployment.yaml', '05-configmap.yaml', '06.hpa.yaml']
                    def yamlDir = 'kubernetes/dev/'

                    // Replace <latest> in dev environment only
                    sh "sed -i 's/<latest>/mfusion-ms-v.1.${BUILD_NUMBER}/g' ${yamlDir}05-deployment.yaml"

                    withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KUBECONFIG'),
                                     [$class: 'AmazonWebServicesCredentialsBinding',
                                      credentialsId: 'aws-credentials',
                                      accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                      secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        yamlFiles.each { yamlFile ->
                            sh """
                                aws configure set aws_access_key_id \$AWS_ACCESS_KEY_ID
                                aws configure set aws_secret_access_key \$AWS_SECRET_ACCESS_KEY
                                aws configure set region ${REGION}

                                kubectl apply -f ${yamlDir}${yamlFile} --kubeconfig=\$KUBECONFIG -n dev --validate=false
                            """
                        }
                    }
                    echo "Deployment to Dev Completed"
                }
            }
        }
    }

    post {
        success {
            echo "Deployment to ${env.BRANCH_NAME} environment completed successfully"
        }
        failure {
            echo "Deployment to ${env.BRANCH_NAME} environment failed. Check logs for details."
        }
    }
}
