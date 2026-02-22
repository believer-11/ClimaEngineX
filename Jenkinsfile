pipeline {
    agent any

    environment {
        // Jenkins Credentials IDs
        DOCKERHUB_CREDS = credentials('dockerhub-creds')
        SONAR_TOKEN     = credentials('sonar-token')
        SONAR_URL       = 'http://<SONARQUBE_IP>:9000'

        DOCKER_IMAGE    = "${DOCKERHUB_CREDS_USR}/knoxweather"
        IMAGE_TAG       = "${BUILD_NUMBER}"
        GITOPS_REPO     = 'https://github.com/<YOUR_GITHUB_USER>/knoxweather-k8s-manifests.git'
    }

    stages {
        stage('Clean Workspace') { steps { cleanWs() } }

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/<YOUR_GITHUB_USER>/knoxweather.git'
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '--scan . --format HTML --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=knoxweather \
                          -Dsonar.projectName=knoxweather \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_URL} \
                          -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL --format table .'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL --format table ${DOCKER_IMAGE}:${IMAGE_TAG}"
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh "echo ${DOCKERHUB_CREDS_PSW} | docker login -u ${DOCKERHUB_CREDS_USR} --password-stdin"
                sh "docker push ${DOCKER_IMAGE}:${IMAGE_TAG}"
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'github-creds', gitToolName: 'Default')]) {
                    sh '''
                        git clone ${GITOPS_REPO} gitops
                        cd gitops

                        # Overwrite image value with new tag
                        sed -i "s|image:.*|image: '"${DOCKER_IMAGE}:${IMAGE_TAG}"'|" deployment.yaml

                        git config user.email "jenkins@knoxcloud.tech"
                        git config user.name "Jenkins CI"
                        git add deployment.yaml
                        git commit -m "Pipeline deployment: ${IMAGE_TAG}"
                        git push "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/<YOUR_GITHUB_USER>/knoxweather-k8s-manifests.git" main
                    '''
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} || true"
            cleanWs()
        }
        success {
            echo '✅ Pipeline completed successfully!'
            mail to: 'rsatale1111@gmail.com',
                 subject: "✅ SUCCESS: Jenkins Pipeline - ${currentBuild.fullDisplayName}",
                 body: "Great news! The KnoxWeather pipeline completed successfully.\n\nProject: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nURL: ${env.BUILD_URL}"
        }
        failure {
            echo '❌ Pipeline failed!'
            mail to: 'rsatale1111@gmail.com',
                 subject: "❌ FAILED: Jenkins Pipeline - ${currentBuild.fullDisplayName}",
                 body: "Attention required! The KnoxWeather pipeline failed.\n\nProject: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nURL: ${env.BUILD_URL}\n\nPlease check the console output for errors."
        }
    }
}
