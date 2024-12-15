pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "879381286690"
        REGION = "ap-south-2"
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
        IMAGE_NAME = "Aniket1402awsdevops/mfusion-ms:mfusion-ms-v.1.${env.BUILD_NUMBER}"
        ECR_IMAGE_NAME = "${ECR_URL}/mfusion-ms:mfusion-ms-v.1.${env.BUILD_NUMBER}"
        KUBECONFIG_ID = 'kubeconfig-aws-aks-k8s-cluster'
        DEV_IMAGE_TAG = "mfusion-ms-v.1.${env.BUILD_NUMBER}"  // Define DEV_IMAGE_TAG
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

                stage('Docker Image Push to Amazon ECR') {
                    steps {
                        echo "Tagging Docker Image for ECR: ${env.ECR_IMAGE_NAME}"
                        sh "docker tag ${env.IMAGE_NAME} ${env.ECR_IMAGE_NAME}"
                        echo "Docker Image Tagging Completed"

                        withDockerRegistry([credentialsId: 'ecr:ap-south-2:aws-ecr-credentials', url: "https://${ECR_URL}"]) {
                            echo "Pushing Docker Image to ECR: ${env.ECR_IMAGE_NAME}"
                            sh "docker push ${env.ECR_IMAGE_NAME}"
                            echo "Docker Image Push to ECR Completed"
                        }
                    }
                }
            }
        }

        stage('Tag Docker Image for Preprod and Prod') {
                    when {
                        anyOf {
                            branch 'preprod'
                            branch 'prod'
                        }
                    }
                    steps {
                        script {
                            def targetTag = BRANCH_NAME == 'preprod' ? PREPROD_IMAGE_TAG : "prod-mfusion-ms-v.1.${BUILD_NUMBER}"
                            def sourceTag = BRANCH_NAME == 'preprod' ? DEV_IMAGE_TAG : PREPROD_IMAGE_TAG
                            def sourceImage = "${ECR_URL}/mfusion-ms:${sourceTag}"
                            def targetImage = "${ECR_URL}/mfusion-ms:${targetTag}"

                            echo "Pulling Source Image: ${sourceImage}"
                            withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                                def pullStatus = sh(script: "docker pull ${sourceImage}", returnStatus: true)
                                if (pullStatus != 0) {
                                    error("Source image ${sourceImage} does not exist or failed to pull.")
                                }
                                echo "Tagging Source Image as Target: ${targetImage}"
                                sh "docker tag ${sourceImage} ${targetImage}"
                                echo "Pushing Target Image to ECR: ${targetImage}"
                                sh "docker push ${targetImage}"
                            }
                            echo "Cleaning Up Local Images"
                            sh "docker rmi ${sourceImage} ${targetImage} || true"
                        }
                    }
                }

        stage('Deploy app to dev env') {
                    when {
                        branch 'dev'
                    }
                    steps {
                        script {
                            echo "Deploying to Dev Environment"
                            def yamlFile = 'Kubernetes/dev/04-deployment.yaml'

                            sh """
                                sed -i 's|<latest>|${DEV_IMAGE_TAG}|g' ${yamlFile}
                                cat ${yamlFile} | grep ${DEV_IMAGE_TAG} || echo "Replacement failed in ${yamlFile}"
                            """
                            sh """
                                kubectl --kubeconfig=/var/lib/jenkins/.kube/config apply -f kubernetes/dev/
                            """

                            def configMapChanged = sh(script: "git diff --name-only HEAD~1 | grep -q 'Kubernetes/dev/05-configmap.yaml'", returnStatus: true)
                            if (configMapChanged == 0) {
                                echo "ConfigMap changed, restarting pods"
                                sh """
                                    kubectl --kubeconfig=/var/lib/jenkins/.kube/config rollout restart deployment dev-mfusion-ms-deployment -n dev
                                """
                            } else {
                                echo "No ConfigMap Changes, Skipping Pod Restart"
                            }
                        }
                    }
                }

        stage('Deploy to Preprod Environment') {
            when {
                branch 'preprod'
            }
            steps {
                script {
                    echo "Deploying to Preprod Environment"
                    def yamlFiles = ['00-ingress.yaml', '02-service.yaml', '03-service-account.yaml', '04-deployment.yaml', '05-configmap.yaml', '06.hpa.yaml']
                    def yamlDir = 'kubernetes/preprod/'

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

                                kubectl apply -f ${yamlDir}${yamlFile} --kubeconfig=\$KUBECONFIG -n preprod --validate=false
                            """
                        }
                    }
                    echo "Deployment to Preprod Completed"
                }
            }
        }

        stage('Deploy to Production Environment') {
            when {
                branch 'prod'
            }
            steps {
                script {
                    echo "Deploying to Prod Environment"
                    def yamlFiles = ['00-ingress.yaml', '02-service.yaml', '03-service-account.yaml', '04-deployment.yaml', '05-configmap.yaml', '06.hpa.yaml']
                    def yamlDir = 'kubernetes/prod/'

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

                                kubectl apply -f ${yamlDir}${yamlFile} --kubeconfig=\$KUBECONFIG -n prod --validate=false
                            """
                        }
                    }
                    echo "Deployment to Prod Completed"
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
