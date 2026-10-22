pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag to build and deploy')
        choice(name: 'SIMULATE_FAULTY_RELEASE', choices: ['NO', 'YES'], description: 'YES deploys a deliberately invalid image to test automatic rollback')
    }

    environment {
        IMAGE_NAME = 'catherine7/smart-campus-energy'
        DEPLOYMENT_NAME = 'campus-energy'
        CONTAINER_NAME = 'energy-dashboard'
        SERVICE_NAME = 'campus-energy-service'
    }

    stages {
        stage('Checkout') {
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
                        powershell -NoProfile -Command "$html = Get-Content index.html -Raw; if ($html -notmatch 'Academic Block' -or $html -notmatch 'Library' -or $html -notmatch 'Research Centre' -or $html -notmatch '100') { exit 1 }"
                        '''
                    }
                }

                stage('Validate Deployment Files') {
                    steps {
                        bat '''
                        if not exist Dockerfile exit /b 1
                        if not exist deployment.yaml exit /b 1
                        if not exist Jenkinsfile exit /b 1
                        findstr /C:"kind: Deployment" deployment.yaml
                        findstr /C:"campus-energy" deployment.yaml
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

        stage('Test Docker Image') {
            steps {
                bat '''
                docker rm -f campus-energy-test 2>NUL
                docker run -d --name campus-energy-test -p 18081:80 %IMAGE_NAME%:%IMAGE_TAG%
                powershell -NoProfile -Command "$ok = $false; for ($i = 0; $i -lt 12; $i++) { try { $r = Invoke-WebRequest -Uri 'http://localhost:18081/' -TimeoutSec 3; if ($r.StatusCode -eq 200 -and $r.Content -match 'Academic Block' -and $r.Content -match 'Research Centre') { $ok = $true; break } } catch { Start-Sleep -Seconds 2 } }; if (-not $ok) { exit 1 }"
                if errorlevel 1 (
                    docker logs campus-energy-test
                    docker rm -f campus-energy-test
                    exit /b 1
                )
                docker rm -f campus-energy-test
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    bat '''
                    docker login -u "%DOCKER_USER%" --password-stdin <<< "%DOCKER_PASS%"
                    '''
                    bat 'docker push %IMAGE_NAME%:%IMAGE_TAG%'
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                        bat 'kubectl apply -f deployment.yaml'

                        try {
                            if (params.SIMULATE_FAULTY_RELEASE == 'YES') {
                                bat 'kubectl set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%IMAGE_NAME%:deliberately-invalid-image'
                            } else {
                                bat 'kubectl set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%IMAGE_NAME%:%IMAGE_TAG%'
                            }

                            bat 'kubectl rollout status deployment/%DEPLOYMENT_NAME% --timeout=90s'

                            bat '''
                            kubectl get deployment %DEPLOYMENT_NAME%
                            kubectl get pods -l app=campus-energy
                            kubectl run campus-energy-check --image=curlimages/curl:latest --restart=Never --rm -i --quiet -- curl -fsS --max-time 10 http://campus-energy-service
                            '''
                        } catch (err) {
                            echo 'Deployment or verification failed. Starting automatic rollback.'

                            bat 'kubectl rollout undo deployment/%DEPLOYMENT_NAME%'
                            bat 'kubectl rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s'

                            bat '''
                            kubectl get deployment %DEPLOYMENT_NAME%
                            kubectl get pods -l app=campus-energy
                            kubectl run campus-energy-recovery-check --image=curlimages/curl:latest --restart=Never --rm -i --quiet -- curl -fsS --max-time 10 http://campus-energy-service
                            '''

                            currentBuild.result = 'UNSTABLE'
                            echo 'Rollback completed and the previous release passed the recovery check.'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f campus-energy-test 2>NUL & exit /b 0'
            echo 'Smart Campus Energy pipeline execution finished.'
        }
        success {
            echo 'Build, image publishing, deployment and verification completed successfully.'
        }
        unstable {
            echo 'A deployment failure occurred. Check the automatic rollback and recovery verification logs.'
        }
        failure {
            echo 'Pipeline failed. Review the failed stage in the Jenkins console output.'
        }
    }
}
