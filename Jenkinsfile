pipeline {
    agent any

    environment {
        // Set up environment variables
        AWS_ACCOUNT_ID = credentials('ACCOUNT_ID')
        AWS_ECR_REPO_NAME = credentials('ECR_REPO_API_GATEWAY')
        AWS_DEFAULT_REGION = 'us-east-1'
        ORGANIZATION_NAME = "fleetman-k8s-ci"
        SERVICE_NAME = "fleetman-api-gateway"
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
                sh 'mvn clean package'
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Image Build') {
            steps {
                script {
                    sh 'docker system prune -f'
                    sh 'docker container prune -f'
                    sh 'docker build -t ${AWS_ECR_REPO_NAME} .'
                }
            }
        }

        stage('ECR Image Pushing') {
            steps {
                script {
                    sh 'aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}'
                    sh 'docker tag ${AWS_ECR_REPO_NAME} ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}'
                    sh 'docker push ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}'
                }
            }
        }

        stage('TRIVY Image Scan') {
            steps {
                sh 'trivy image ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER} > trivyimage.txt'
            }
        }

        stage('Update Deployment File') {
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    script {
                        def ecrRepoName = env.AWS_ECR_REPO_NAME
                        def buildNumber = env.BUILD_NUMBER

                        sh '''
                            git config user.email "dksasi77@gmail.com"
                            git config user.name "DKSASI2003"
                            echo "Build number: ${buildNumber}"
                            
                            # Display the content of deploy.yaml for debugging
                            cat deploy.yaml

                            # Extract the current image tag
                            imageTag=$(grep -oP '(?<=fleetman-position-simulator:)[^ ]+' deploy.yaml || echo "")
                            echo "Current image tag: $imageTag"
                            
                            if [ -z "$imageTag" ]; then
                                echo "Error: Image tag not found in deploy.yaml"
                                exit 1
                            fi

                            # Update the image tag
                            sed -i "s/${ecrRepoName}:${imageTag}/${ecrRepoName}:${buildNumber}/" deploy.yaml
                            
                            git add deploy.yaml
                            git commit -m "Update deployment Image to version ${buildNumber}"
                            
                            # Pull latest changes to ensure the local branch is up-to-date
                            git pull --rebase origin master

                            # Push changes
                            git push https://${GITHUB_TOKEN}@github.com/${env.ORGANIZATION_NAME}/${env.SERVICE_NAME}.git HEAD:master
                        '''
                    }
                }
            }
        }
    }
}
