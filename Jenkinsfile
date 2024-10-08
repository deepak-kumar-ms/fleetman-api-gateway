pipeline {
    agent any

    environment {
        // You must set the following environment variables
        AWS_ACCOUNT_ID = credentials('ACCOUNT_ID')
        AWS_ECR_REPO_NAME = credentials('ECR_REPO_API_GATEWAY')
        AWS_DEFAULT_REGION = 'us-east-1'
        ORGANIZATION_NAME = "deepak-kumar-ms"
        SERVICE_NAME = "fleetman-api-gateway"
        GITHUB_PASS = 'gitganesh007'
            
        REPOSITORY_TAG = "${ORGANIZATION_NAME}-${SERVICE_NAME}:${BUILD_ID}"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/"
    }

    stages {
      stage('Preparation') {
         steps {
            cleanWs()
         }
      }
      stage('Checkout from Git') {
            steps {
                git credentialsId: 'GitHub', url: "https://github.com/${ORGANIZATION_NAME}/${SERVICE_NAME}"
            }
      }

      stage('Build') {
         steps {
            sh '''mvn clean package'''
         }
      }

      stage('Trivy File Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

      stage("Docker Image Build") {
            steps {
                script {
                    sh 'docker system prune -f'
                    sh 'docker container prune -f'
                    sh 'docker build -t ${AWS_ECR_REPO_NAME} .'
                }
            }
        }

      stage("ECR Image Pushing") {
            steps {
                script {
                        sh 'aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}'
                        sh 'docker tag ${AWS_ECR_REPO_NAME} ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}'
                        sh 'docker push ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}'
                }
            }
        }

      stage("TRIVY Image Scan") {
            steps {
                sh 'trivy image ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER} > trivyimage.txt'
            }
        }

      stage('Checkout Code') {
            steps {
                git credentialsId: 'GitHub', url: "https://github.com/${ORGANIZATION_NAME}/${SERVICE_NAME}"
            }
        }

      stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "fleetman-position-simulator"
                GIT_ORG_NAME = "deepak-kumar-ms"
            }
            stage('Update Deployment file') {
            steps {
                sh """
                    git config user.email "dksasi77@gmail.com"
                    git config user.name "DKSASI2003"
                    
                    # Your deployment file update logic here
                    # For example:
                    # sed -i 's/old-tag/new-tag/' deploy.yaml
                    
                    # Commit and push changes
                    git add deploy.yaml
                    git commit -m "Update deployment file"
                    git push https://${GITHUB_USER}:${GITHUB_PASS}@github.com/${GIT_ORG_NAME}/${GIT_REPO_NAME} master
                """
            }
        }
        }
    }
}
