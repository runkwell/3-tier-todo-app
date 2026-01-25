pipeline {
    agent {
        label 'Jenkins-Bastion-Runner'
    }

    environment {
        AWS_REGION = "ap-southeast-1"
        DEPLOY_DIR = "/opt/todo-app"
        IMAGE_TAG = "${env.GIT_COMMIT.take(8)}"
    }

    

    stages {
        stage('Prepare Environment') {
            steps {
                script {
                    def accountId = sh(script: "aws sts get-caller-identity --query Account --output text", returnStdout: true).trim()
                    env.ECR_REPO = "${accountId}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    env.ECR_BACKEND = "${env.ECR_REPO}/backend"
                    env.ECR_FRONTEND = "${env.ECR_REPO}/frontend"
                    
                    echo "Running with Account ID: ${accountId}"
                    echo "ECR Repo: ${env.ECR_REPO}"
                }
            }
        }



        stage('Build & Push to ECR') {
            parallel {
                stage('Backend Build') {
                    steps {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${env.ECR_REPO}
                            docker build -t ${env.ECR_BACKEND}:${IMAGE_TAG} -f ./flask/Dockerfile flask/
                            docker push ${env.ECR_BACKEND}:${IMAGE_TAG}
                        """
                    }
                }
                stage('Frontend Build') {
                    steps {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${env.ECR_REPO}
                            docker build -t ${env.ECR_FRONTEND}:${IMAGE_TAG} -f ./react/Dockerfile react/
                            docker push ${env.ECR_FRONTEND}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        # Render template
                        cat ./user_data.sh.tmpl \
                          | sed "s|{{IMAGE_TAG}}|${IMAGE_TAG}|g" \
                          | sed "s|{{ECR_REPO}}|${ECR_REPO}|g" \
                          | sed "s|{{DEPLOY_DIR}}|${DEPLOY_DIR}|g" \
                          | sed "s|{{ECR_BACKEND}}|${ECR_BACKEND}|g" \
                          | sed "s|{{ECR_FRONTEND}}|${ECR_FRONTEND}|g" \
                          | sed "s|{{SQLALCHEMY_DATABASE_URI}}|${env.SQLALCHEMY_DATABASE_URI}|g" \
                          | sed "s|{{JWT_SECRET_KEY}}|${env.JWT_SECRET_KEY}|g" \
                          > user-data.sh

                        # Base64 encode
                        USER_DATA_B64=\$(base64 < user-data.sh | tr -d '\\n')

                        # Get the golden launch template data
                        LT_DATA=\$(aws ec2 describe-launch-template-versions \
                          --launch-template-id "${LT_VERSION}" \
                          --versions 34 \
                          --query 'LaunchTemplateVersions[0].LaunchTemplateData' \
                          --output json)

                        # Add new UserData to LaunchTemplateData (yêu cầu 'jq' đã cài trên node)
                        LT_DATA_WITH_UD=\$(echo "\$LT_DATA" | jq --arg ud "\$USER_DATA_B64" '.UserData = \$ud')

                        # Create new launch template version
                        NEW_VERSION=\$(aws ec2 create-launch-template-version \
                          --launch-template-id "${LT_VERSION}" \
                          --launch-template-data "\$LT_DATA_WITH_UD" \
                          --query 'LaunchTemplateVersion.VersionNumber' \
                          --output text)

                        echo "Created launch template version \$NEW_VERSION"

                        echo "Starting instance refresh..."
                        aws autoscaling start-instance-refresh \
                          --auto-scaling-group-name "${ASG_NAME}" \
                          --preferences '{"MinHealthyPercentage":80}' \
                          --strategy Rolling
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished."
        }
    }
}