// Place this file at the ROOT of the tshirt-shop-frontend repo, replacing
// whatever is there now. This repo IS the service - so docker build context
// is "." not "./frontend" (that only applied to the old single-repo layout).

pipeline {
    agent any

    environment {
        SERVICE_NAME = 'frontend'
        DOCKERHUB_NAMESPACE = 'devivek'   // your actual Docker Hub namespace
        IMAGE = "${DOCKERHUB_NAMESPACE}/tshirt-shop-${SERVICE_NAME}"
        IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
        EC2_HOST = 'ubuntu@your-ec2-public-ip'   // <-- fill in your real EC2 public IP
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonarqube-server') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=tshirt-shop-frontend -Dsonar.sources=./public"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t ${IMAGE}:${IMAGE_TAG} -t ${IMAGE}:latest ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push ${IMAGE}:${IMAGE_TAG}
                        docker push ${IMAGE}:latest
                    '''
                }
            }
        }

        // --- DEV: auto-deploys on every push to `dev` branch ---
        stage('Deploy to Dev') {
            when { branch 'dev' }
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no $EC2_HOST "
                          docker pull ${IMAGE}:${IMAGE_TAG} &&
                          docker rm -f ${SERVICE_NAME}-dev || true &&
                          docker run -d --name ${SERVICE_NAME}-dev -p 127.0.0.1:9500:80 ${IMAGE}:${IMAGE_TAG}
                        "
                    '''
                }
            }
        }

        // --- STAGING: auto-deploys on every push to `staging` branch ---
        stage('Deploy to Staging') {
            when { branch 'staging' }
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no $EC2_HOST "
                          docker pull ${IMAGE}:${IMAGE_TAG} &&
                          docker rm -f ${SERVICE_NAME}-staging || true &&
                          docker run -d --name ${SERVICE_NAME}-staging -p 127.0.0.1:9000:80 ${IMAGE}:${IMAGE_TAG}
                        "
                    '''
                }
            }
        }

        // --- PRODUCTION: only from `main`, requires manual approval, zero-downtime blue-green ---
        stage('Approval Gate') {
            when { branch 'main' }
            steps {
                input message: "Deploy frontend:${IMAGE_TAG} to PRODUCTION?", ok: 'Deploy'
            }
        }

        stage('Deploy to Production (blue-green)') {
            when { branch 'main' }
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no scripts/deploy-blue-green.sh $EC2_HOST:/home/ubuntu/tshirt-shop/deploy-blue-green.sh || true
                        ssh -o StrictHostKeyChecking=no $EC2_HOST "
                          chmod +x /home/ubuntu/tshirt-shop/deploy-blue-green.sh &&
                          /home/ubuntu/tshirt-shop/deploy-blue-green.sh ${SERVICE_NAME} ${IMAGE} ${IMAGE_TAG} 8000 8001
                        "
                    '''
                }
            }
        }
    }

    post {
        failure { echo "Pipeline failed for frontend - deploy stage never ran." }
        always  { sh 'docker logout || true' }
    }
}

