pipeline {
    agent any
    
    tools {
        maven 'M3'
        jdk 'JDK11'
    }
    
    environment {
        MAVEN_SETTINGS = '/var/lib/jenkins/.m2/settings.xml'
        AWS_REGION = 'ap-south-1'
        // ECR_REPO = credentials('ECR_REPO')
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "13.234.247.67:8081"
        NEXUS_REPOSITORY = "boardgame_repo"
        NEXUS_CREDENTIAL_ID = "nexus-cred"
        ARTIFACT_VERSION = "${BUILD_NUMBER}"
        EKS_CLUSTER="boardgame-eks-cluster"
        ECR_REGISTRY="123115277439.dkr.ecr.ap-south-1.amazonaws.com"
        IMAGE_TAG="latest"
        ECR_REPO="my-app"
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
                timeout(time: 2, unit: 'MINUTES') {
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
        // stage('Build Docker Image') {
        //     steps {
        //         echo "🐳 Starting Docker build for image: my-app:build${BUILD_NUMBER}"
        //         sh "docker build -t my-app:build${BUILD_NUMBER} ."
        //         echo "✅ Docker build completed for image: my-app:build${BUILD_NUMBER}"
        //     }
        // }
        
        stage('Docker Build') {
            steps {
                sh """
                  docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG .
                """
            }
        }

        
        stage('Scan Docker Image with Trivy') {
            steps {
                echo "🔍 Starting Trivy scan for Docker image: my-app:build${BUILD_NUMBER}"
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
        
        // stage('Login to AWS ECR') {
        //     steps {
        //         echo "🔑 Logging in to AWS ECR..."
        //         sh '''
        //             aws ecr get-login-password --region ${AWS_REGION} | \
        //             docker login --username AWS --password-stdin ${ECR_REGISTRY}
        //             '''
        //         echo "✅ AWS ECR login successful"
        //     }
        // }

    //   stage('Push Docker Image to ECR') {
    //         steps {
    //             echo "🚀 Tagging and pushing Docker image to ECR: ${ECR_REPO}:${BUILD_NUMBER}"
    //             sh """
    //               docker tag my-app:build${BUILD_NUMBER} ${ECR_REPO}:${BUILD_NUMBER}
    //               docker push ${ECR_REPO}:${BUILD_NUMBER}
    //             """
    //             echo "✅ Docker image pushed: ${ECR_REPO}:${BUILD_NUMBER}"
    //         }
    //     }
        stage('Push to ECR') {
            steps {
                sh "docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG"
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    // Update kubeconfig
                    sh "aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER"
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }
    }
    
    post {
        failure {
            mail to: 'lakshitladha05@gmail.com',
                subject: "❌ FAILED: ${env.JOB_NAME} [#${env.BUILD_NUMBER}]",
                body: """\
                Hello Team,
                
                The build for *${env.JOB_NAME}* has **FAILED**.
                
                🔢 Build Number : ${env.BUILD_NUMBER}
                👤 Triggered By : ${env.BUILD_USER ?: 'N/A'}
                
                📄 Console Log : ${env.BUILD_URL}
                🌊 Blue Ocean   : ${env.RUN_DISPLAY_URL}
                
                Please check the logs and fix the issue.
                """
        }
    
        success {
            mail to: 'lakshitladha05@gmail.com',
                subject: "✅ SUCCESS: ${env.JOB_NAME} [#${env.BUILD_NUMBER}]",
                body: """\
                Hello Team,
                
                The build for *${env.JOB_NAME}* was **SUCCESSFUL**.
                
                🔢 Build Number : ${env.BUILD_NUMBER}
                👤 Triggered By : ${env.BUILD_USER ?: 'N/A'}
                
                📄 Console Log : ${env.BUILD_URL}
                🌊 Blue Ocean   : ${env.RUN_DISPLAY_URL}
                """
        }
    }

}
