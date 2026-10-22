
pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
        booleanParam(name: 'PUSH_IMAGE', defaultValue: false, description: 'Push image and deploy to Kubernetes')
    }

    environment {
        IMAGE_NAME = 'catherine7/smart-campus-energy'
    }

    stages {
        stage('Checkout Latest Code') {
            steps {
                checkout scm
                bat 'git log -1 --oneline'
            }
        }

        stage('Parallel Tests') {
            parallel {
                stage('Test Dashboard') {
                    steps {
                        bat '''
                        if not exist index.html exit /b 1
                        findstr /C:"Campus Energy Summary" index.html
                        if errorlevel 1 exit /b 1
                        findstr /C:"Academic Block" index.html
                        if errorlevel 1 exit /b 1
                        findstr /C:"Library" index.html
                        if errorlevel 1 exit /b 1
                        findstr /C:"Research Centre" index.html
                        if errorlevel 1 exit /b 1
                        findstr /C:"totalUsage" index.html
                        if errorlevel 1 exit /b 1
                        findstr /C:"highestUsage" index.html
                        if errorlevel 1 exit /b 1
                        '''
                    }
                }

                stage('Validate Docker and Kubernetes Files') {
                    steps {
                        bat '''
                        if not exist Dockerfile exit /b 1
                        if not exist deployment.yaml exit /b 1
                        findstr /C:"FROM nginx:alpine" Dockerfile
                        if errorlevel 1 exit /b 1
                        findstr /C:"kind: Deployment" deployment.yaml
                        if errorlevel 1 exit /b 1
                        '''
                    }
                }
            }
        }

        stage('Archive Build Artifacts') {
            steps {
                archiveArtifacts artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile', fingerprint: true
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'
            }
        }

        stage('Test Application Container') {
            steps {
                script {
                    try {
                        bat 'docker rm -f campus-energy-test 2>NUL & exit /b 0'
                        bat 'docker run -d --name campus-energy-test -p 18081:80 %IMAGE_NAME%:%IMAGE_TAG%'
                        bat 'powershell -NoProfile -Command "Start-Sleep -Seconds 3"'
                        bat '''
                        curl.exe -fsS http://localhost:18081/ -o dashboard-response.html
                        if errorlevel 1 exit /b 1
                        findstr /C:"Campus Energy Summary" dashboard-response.html
                        if errorlevel 1 exit /b 1
                        findstr /C:"Academic Block" dashboard-response.html
                        if errorlevel 1 exit /b 1
                        findstr /C:"Research Centre" dashboard-response.html
                        if errorlevel 1 exit /b 1
                        '''
                    } finally {
                        bat 'docker rm -f campus-energy-test 2>NUL & exit /b 0'
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            when {
                expression { return params.PUSH_IMAGE }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat '''
                    powershell -NoProfile -Command "$env:DOCKER_PASS | docker login -u $env:DOCKER_USER --password-stdin"
                    if errorlevel 1 exit /b 1
                    docker push %IMAGE_NAME%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { return params.PUSH_IMAGE }
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG_FILE')]) {
                    script {
                        bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" apply -f deployment.yaml
                        if errorlevel 1 exit /b 1
                        '''
                        bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/campus-energy energy-dashboard=%IMAGE_NAME%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                        '''
                        script {
                            def rolloutResult = bat(
                                returnStatus: true,
                                script: 'kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/campus-energy --timeout=120s'
                            )

                            if (rolloutResult != 0) {
                                echo 'Deployment failed. Attempting automatic rollback.'

                                bat '''
                                kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout undo deployment/campus-energy
                                if errorlevel 1 exit /b 1
                                kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/campus-energy --timeout=120s
                                if errorlevel 1 exit /b 1
                                '''

                                error('Deployment failed; previous version was restored.')
                            }
                        }

                        bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get deployments
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get pods
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get services
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Dashboard tested, Docker image built, and requested deployment completed.'
        }
        failure {
            echo 'FAILED: Check Console Output for the failed stage.'
        }
        always {
            echo 'Smart Campus Energy pipeline finished.'
        }
    }
}
