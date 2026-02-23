pipeline {
    agent any

    environment {
        // Jenkins Credentials IDs
        DOCKERHUB_CREDS = credentials('dockerhub-creds')
        SONAR_TOKEN     = credentials('sonar-token')
        SONAR_URL       = 'http://13.221.244.76:9000'

        DOCKER_IMAGE    = "${DOCKERHUB_CREDS_USR}/knoxweather"
        IMAGE_TAG       = "${BUILD_NUMBER}"
        GITOPS_REPO     = 'https://github.com/believer-11/ClimaEngineX.git'
    }

    stages {
        stage('Clean Workspace') { steps { cleanWs() } }

                stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [
                        [$class: 'MessageExclusion', excludedMessage: '(?s).*?\\[ci skip\\].*']
                    ],
                    userRemoteConfigs: [[url: "${GITOPS_REPO}"]]
                ])
            }
        }

        stage('Check for CI Skip') {
            steps {
                script {
                    def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    if (commitMessage.contains('[ci skip]')) {
                        currentBuild.result = 'NOT_BUILT'
                        error('Build skipped due to [ci skip] in commit message')
                    }
                }
            }
        }

                stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: ' --scan ./', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }


                stage('SonarQube Analysis') {
            environment {
                SCANNER_HOME = tool 'sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
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

        stage('Update K8s Manifests') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'github-creds', gitToolName: 'Default')]) {
                    sh '''
                        # Go into the new k8s folder you created
                        cd k8s

                        # Overwrite image value with new tag
                        sed -i "s|image:.*|image: '"${DOCKER_IMAGE}:${IMAGE_TAG}"'|" deployment.yaml

                        git config user.email "jenkins@knoxcloud.tech"
                        git config user.name "Jenkins CI"
                        
                        # Commit the change directly back to this repo
                        git add deployment.yaml
                        # Use [ci skip] in the commit message so GitHub doesn't trigger another Jenkins build by accident!
                        git commit -m "Pipeline deployment: ${IMAGE_TAG} [ci skip]"
                        
                        git push "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/believer-11/ClimaEngineX.git" main
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
                 body: "Great news! The KnoxWeather pipeline completed successfully.\\n\\nProject: ${env.JOB_NAME}\\nBuild Number: ${env.BUILD_NUMBER}\\nURL: ${env.BUILD_URL}"
        }
        failure {
            echo '❌ Pipeline failed!'
            mail to: 'rsatale1111@gmail.com',
                 subject: "❌ FAILED: Jenkins Pipeline - ${currentBuild.fullDisplayName}",
                 body: "Attention required! The KnoxWeather pipeline failed.\\n\\nProject: ${env.JOB_NAME}\\nBuild Number: ${env.BUILD_NUMBER}\\nURL: ${env.BUILD_URL}\\n\\nPlease check the console output for errors."
        }
    }
}
