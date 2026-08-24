pipeline {

    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    environment {
        OWNER = 'jiyajagiasi'
        IMAGE_NAME = 'notes-application'
        GITOPS_REPO = 'notes-gitops'
        MANIFEST = 'apps/notes-api/deployment.yaml'

        IMAGE = "ghcr.io/${OWNER}/${IMAGE_NAME}"

        TEST_PORT = '18087'
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    def scmVars = checkout scm
                    env.SHA = scmVars.GIT_COMMIT
                }

                echo "Building ${env.IMAGE}:${env.SHA}"
            }
        }

        stage('Build') {
            steps {
                bat '''
                    docker build -t "%IMAGE%:%SHA%" -t "%IMAGE%:latest" .
                '''
            }
        }

        stage('Smoke test') {
            steps {
                powershell '''
                    docker rm -f smoke 2>$null

                    docker run -d --name smoke -p "${env:TEST_PORT}:8080" "${env:IMAGE}:${env:SHA}"

                    Write-Host "Waiting for application..."

                    for ($i = 1; $i -le 20; $i++) {

                        try {
                            $response = Invoke-WebRequest `
                                -Uri "http://localhost:${env:TEST_PORT}/healthz" `
                                -UseBasicParsing `
                                -TimeoutSec 2

                            if ($response.StatusCode -eq 200) {
                                Write-Host "Application is healthy."
                                break
                            }
                        }
                        catch {
                            Write-Host "Waiting..."
                            Start-Sleep -Seconds 1
                        }
                    }

                    Invoke-WebRequest `
                        -Uri "http://localhost:${env:TEST_PORT}/healthz" `
                        -UseBasicParsing

                    Invoke-WebRequest `
                        -Uri "http://localhost:${env:TEST_PORT}/ready" `
                        -UseBasicParsing
                '''
            }

            post {
                always {
                    bat '''
                        docker rm -f smoke >nul 2>&1
                        exit /b 0
                    '''
                }
            }
        }

        stage('Push to GHCR') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'ghcr-credentials',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_PAT'
                    )
                ]) {

                    bat '''
                        echo %GHCR_PAT% | docker login ghcr.io -u %GHCR_USER% --password-stdin

                        docker push "%IMAGE%:%SHA%"

                        docker logout ghcr.io
                    '''
                }
            }
        }

        stage('Deploy: write the tag into notes-gitops') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'gitops-token',
                        usernameVariable: 'GITOPS_USER',
                        passwordVariable: 'GITOPS_TOKEN'
                    )
                ]) {

                    powershell '''
                        if (Test-Path "gitops") {
                            Remove-Item -Recurse -Force "gitops"
                        }

                        git clone --depth 1 `
                            "https://github.com/$env:OWNER/$env:GITOPS_REPO.git" `
                            gitops

                        Set-Location gitops

                        $manifest = $env:MANIFEST

                        $content = Get-Content $manifest -Raw

                        $newContent = $content -replace `
                            'image:\\s*ghcr\\.io/[^\\s]+', `
                            "image: $env:IMAGE`:$env:SHA"

                        Set-Content `
                            -Path $manifest `
                            -Value $newContent `
                            -NoNewline

                        git config user.name "jenkins-ci"
                        git config user.email "jenkins-ci@users.noreply.github.com"

                        git add $manifest

                        $changes = git diff --cached --quiet

                        if ($LASTEXITCODE -eq 0) {
                            Write-Host "Image tag unchanged - nothing to deploy"
                            exit 0
                        }

                        git commit -m "deploy $env:IMAGE_NAME`:$env:SHA"

                        $pushUrl = "https://$env:GITOPS_USER`:$env:GITOPS_TOKEN@github.com/$env:OWNER/$env:GITOPS_REPO.git"

                        git push $pushUrl HEAD:main

                        Write-Host "Updated GitOps repository."
                        Write-Host "Argo CD will pick up the change."
                    '''
                }
            }
        }
    }

    post {

        success {
            echo "Deployed ${env.IMAGE}:${env.SHA} - Argo CD takes it from here."
        }

        failure {
            echo 'FAILED - check the red stage in Stage View.'
        }

        always {
            bat '''
                docker image prune -f >nul 2>&1
                exit /b 0
            '''

            cleanWs()
        }
    }
}