pipeline {
    agent any
    
    tools {
        maven 'M3'
        jdk 'JDK11'
    }
    
    environment {
        MAVEN_SETTINGS = '/var/lib/jenkins/.m2/settings.xml'
        AWS_REGION = 'ap-south-1'
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "13.234.247.67:8081"
        NEXUS_REPOSITORY = "boardgame_repo"
        NEXUS_CREDENTIAL_ID = "nexus-cred"
        ARTIFACT_VERSION = "${BUILD_NUMBER}"
        EKS_CLUSTER="boardgame-eks-cluster"
        ECR_REGISTRY="123115277439.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_NAME="boardgame-project-image"
        IMAGE_TAG="latest"
        ECR_REPO="my-app"
        AWS_ACCOUNT_NUMER="123115277439"
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'PAT-for-Github-Auth',
                        url: 'https://github.com/Lakshitladha007/Boardgame_Project.git'
                    ]]
                )
            }
        }
        
        stage('Code Analysis') {
            environment {
                scannerHome = tool 'My_SonarQube'  
            }
            steps {
                script {
                    withSonarQubeEnv('My_SonarQube') {
                        withCredentials([string(credentialsId: 'SonarQube_PAT', variable: 'SONAR_TOKEN')]) {
                            sh "mvn clean install sonar:sonar \
                                -DskipTests \
                                -Dsonar.projectKey=boardgame-app \
                                -Dsonar.projectName=\"Boardgame Project\" \
                                -Dsonar.projectVersion=${BUILD_NUMBER} \
                                -Dsonar.sources=src/main/java \
                                -Dsonar.login=$SONAR_TOKEN"
                        }
                    }
                }
            }
        }
        
        stage("Quality Gate") {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
        
        stage('Deploy to Nexus') {
            steps {
                echo "📦 Deploying artifact to Nexus..."
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', 
                                                  usernameVariable: 'NEXUS_USER', 
                                                  passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        mvn deploy -DskipTests=true \
                                   -Dnexus.username=${NEXUS_USER} \
                                   -Dnexus.password=${NEXUS_PASS} \
                                   -s ${MAVEN_SETTINGS}
                    """
                }
                echo "✅ Artifact deployment completed"
            }
        }
        
        stage('Docker Build') {
            steps {
                sh """
                  docker build -t ${IMAGE_NAME}:${env.BUILD_NUMBER} .
                """
            }
        }

        
        stage('Scan Docker Image with Trivy') {
            steps {
                echo "🔍 Starting Trivy scan for Docker image: ${IMAGE_NAME}:${env.BUILD_NUMBER} "
                sh """
                  trivy image --severity HIGH,CRITICAL -f table -o trivy-report.txt my-app:build${BUILD_NUMBER} || true
                """
                echo "✅ Trivy scan completed. Report archived as trivy-report.txt"
                archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
            }
        }
        
        stage('Login to AWS ECR') {
            steps {
                echo "🔑 Logging in to AWS ECR..."
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    '''
                }
                echo "✅ AWS ECR login successful"
            }
        }
      
        stage('Push docker image to ECR') {
            steps {
                sh """
                docker tag ${IMAGE_NAME}:${env.BUILD_NUMBER} ${AWS_ACCOUNT_NUMER}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${env.BUILD_NUMBER}
                docker push ${AWS_ACCOUNT_NUMER}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${env.BUILD_NUMBER}"""
            }
            post {
                success {
                    sh """
                        docker rmi ${IMAGE_NAME}:${env.BUILD_NUMBER} || true
                    """
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    sh """
                       sed -e "s|ECR_REGISTRY|${ECR_REGISTRY}|g" \
                           -e "s|ECR_REPO|${ECR_REPO}|g" \
                           -e "s|BUILD_ID|${env.BUILD_NUMBER}|g" \
                           deployment-service.yaml > deployment.yaml
                    """
                }
                script {
                    sh 'cat deployment.yaml'
                    sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
                    sh 'kubectl apply -f deployment.yaml'
                }
            }
        }
    }
    
    post {
        failure {
            mail to: 'lakshitladha05@gmail.com',
                subject: "FAILED: ${env.JOB_NAME} [#${env.BUILD_NUMBER}]",
                body: """\
                Hello Team,
                
                The build for *${env.JOB_NAME}* has **FAILED**.
                
                Build Number : ${env.BUILD_NUMBER}
                
                Console Log : ${env.BUILD_URL}
                Blue Ocean   : ${env.RUN_DISPLAY_URL}
                
                Please check the logs and fix the issue.
                """
        }
    
        success {
            mail to: 'lakshitladha05@gmail.com',
                subject: "SUCCESS: ${env.JOB_NAME} [#${env.BUILD_NUMBER}]",
                body: """\
                Hello Team,
                
                The build for *${env.JOB_NAME}* was **SUCCESSFUL**.
                
                Build Number : ${env.BUILD_NUMBER}
                
                Console Log : ${env.BUILD_URL}
                Blue Ocean   : ${env.RUN_DISPLAY_URL}
                """
        }
    }
}