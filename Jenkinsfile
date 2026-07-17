pipeline {
    agent any

    environment {
        DOCKERHUB_NAMESPACE = 'your_dockerhub_username'
        IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
        EC2_HOST = 'ec2-user@your-ec2-public-ip'   // adjust user for your AMI (ubuntu/ec2-user)
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') { // name configured in Jenkins > System
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=tshirt-shop \
                          -Dsonar.sources=./api/src,./admin/src,./frontend/public
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                // Fails the build if SonarQube quality gate fails - don't skip this,
                // a scan nobody enforces is just noise
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Images') {
            steps {
                sh '''
                    docker build -t $DOCKERHUB_NAMESPACE/tshirt-shop-frontend:$IMAGE_TAG -t $DOCKERHUB_NAMESPACE/tshirt-shop-frontend:latest ./frontend
                    docker build -t $DOCKERHUB_NAMESPACE/tshirt-shop-api:$IMAGE_TAG -t $DOCKERHUB_NAMESPACE/tshirt-shop-api:latest ./api
                    docker build -t $DOCKERHUB_NAMESPACE/tshirt-shop-admin:$IMAGE_TAG -t $DOCKERHUB_NAMESPACE/tshirt-shop-admin:latest ./admin
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $DOCKERHUB_NAMESPACE/tshirt-shop-frontend:$IMAGE_TAG
                        docker push $DOCKERHUB_NAMESPACE/tshirt-shop-frontend:latest
                        docker push $DOCKERHUB_NAMESPACE/tshirt-shop-api:$IMAGE_TAG
                        docker push $DOCKERHUB_NAMESPACE/tshirt-shop-api:latest
                        docker push $DOCKERHUB_NAMESPACE/tshirt-shop-admin:$IMAGE_TAG
                        docker push $DOCKERHUB_NAMESPACE/tshirt-shop-admin:latest
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                // SSH key stored as a Jenkins credential, never in the repo.
                // Copies compose file, updates image tag, pulls, restarts.
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no docker-compose.prod.yml $EC2_HOST:/home/ec2-user/tshirt-shop/docker-compose.yml
                        ssh -o StrictHostKeyChecking=no $EC2_HOST "cd /home/ec2-user/tshirt-shop && \
                            sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=$IMAGE_TAG/' .env && \
                            docker compose pull && \
                            docker compose up -d && \
                            docker image prune -f"
                    '''
                }
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed - check the stage logs above. Deploy stage never runs if build/scan/push failed.'
        }
        always {
            sh 'docker logout || true'
        }
    }
}
