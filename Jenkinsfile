pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
        booleanParam(name: 'PUSH_IMAGE', defaultValue: false, description: 'Push image to Docker Hub')
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
                        powershell -NoProfile -Command "$h = Get-Content index.html -Raw; if ($h -notmatch 'Academic Block' -or $h -notmatch 'Library' -or $h -notmatch 'Research Centre' -or $h -notmatch 'totalUsage' -or $h -notmatch 'highestUsage') { exit 1 }"
                        '''
                    }
                }

                stage('Validate Docker and Kubernetes Files') {
                    steps {
                        bat '''
                        if not exist Dockerfile exit /b 1
                        if not exist deployment.yaml exit /b 1
                        findstr /C:"FROM nginx:alpine" Dockerfile
                        findstr /C:"kind: Deployment" deployment.yaml
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
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Latest code was tested, built and archived.'
        }
        failure {
            echo 'FAILED: Check Console Output for the failed stage.'
        }
        always {
            echo 'Smart Campus Energy pipeline finished.'
        }
    }
}
