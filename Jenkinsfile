
pipeline {
    agent any

    environment {
        IMAGE = 'niharikarao15/online-bookstore'
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"
                    docker build -t $image .
                    if ($LASTEXITCODE -ne 0) { exit 1 }
                '''
            }
        }

        stage('Automated Test Gate') {
            steps {
                powershell '''
                    $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"
                    docker rm -f bookstore-ci-test 2>$null
                    docker run -d --name bookstore-ci-test $image
                    if ($LASTEXITCODE -ne 0) { throw "Container startup failed" }

                    try {
                        docker exec bookstore-ci-test sh -c "wget -q -O /dev/null http://127.0.0.1/"
                        if ($LASTEXITCODE -ne 0) { throw "Website smoke test failed" }

                        docker exec bookstore-ci-test sh -c "grep -q 'Clean Code' /usr/share/nginx/html/index.html"
                        if ($LASTEXITCODE -ne 0) { throw "Book listing test failed" }

                        docker exec bookstore-ci-test sh -c "grep -q 'Atomic Habits' /usr/share/nginx/html/index.html"
                        if ($LASTEXITCODE -ne 0) { throw "Required book missing" }

                        Write-Host "PASS: HTTP response and required books verified"
                    }
                    finally {
                        docker rm -f bookstore-ci-test
                    }
                '''
            }
        }

        stage('Security Gate') {
            steps {
                powershell '''
                    docker run --rm --entrypoint sh $env:IMAGE:latest -c "test -f /usr/share/nginx/html/index.html"
                    if ($LASTEXITCODE -ne 0) { throw "Security/pre-deployment gate failed" }
                    Write-Host "PASS: Required application file exists"
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    powershell '''
                        $env:DOCKERHUB_TOKEN | docker login -u $env:DOCKERHUB_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) { throw "Docker Hub login failed" }

                        $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"
                        docker push $image
                        if ($LASTEXITCODE -ne 0) { throw "Docker push failed" }

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    powershell '''
                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) { throw "Kubernetes apply failed" }

                        $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"
                        kubectl set image deployment/online-bookstore "bookstore=$image"
                        if ($LASTEXITCODE -ne 0) { throw "Image update failed" }

                        kubectl rollout status deployment/online-bookstore --timeout=120s
                        if ($LASTEXITCODE -ne 0) {
                            Write-Host "Rollout failed. Rolling back."
                            kubectl rollout undo deployment/online-bookstore
                            kubectl rollout status deployment/online-bookstore --timeout=120s
                            throw "Deployment failed; rollback attempted"
                        }
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Online Bookstore CI/CD pipeline finished. Review stage results.'
        }
    }
}
