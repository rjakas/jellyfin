// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  serviceAccountName: jenkins
  containers:
  - name: dotnet
    image: mcr.microsoft.com/dotnet/sdk:8.0
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    environment {
        DOTNET_CLI_HOME = "/tmp/dotnet"
        DOTNET_SKIP_FIRST_TIME_EXPERIENCE = "true"
        DOTNET_NOLOGO = "true"
        
        // Docker registry
        DOCKER_REGISTRY = "docker.io"  // CUSTOMIZE: your registry
        DOCKER_IMAGE = "your-username/jellyfin"  // CUSTOMIZE: your image name
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        
        // Credentials
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "=== Checking out code from GitHub ==="
                    checkout scm
                    
                    // Get commit info
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    
                    env.GIT_BRANCH = sh(
                        script: "git rev-parse --abbrev-ref HEAD",
                        returnStdout: true
                    ).trim()
                    
                    echo "Branch: ${env.GIT_BRANCH}"
                    echo "Commit: ${env.GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('Restore Dependencies') {
            steps {
                container('dotnet') {
                    script {
                        echo "=== Restoring NuGet packages ==="
                        sh '''
                            dotnet restore Jellyfin.sln
                        '''
                    }
                }
            }
        }

        stage('Build') {
            steps {
                container('dotnet') {
                    script {
                        echo "=== Building solution ==="
                        sh '''
                            dotnet build Jellyfin.sln \
                                --configuration Release \
                                --no-restore \
                                --verbosity minimal
                        '''
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                container('dotnet') {
                    script {
                        echo "=== Running unit tests ==="
                        sh '''
                            # Run all test projects
                            dotnet test Jellyfin.sln \
                                --configuration Release \
                                --no-build \
                                --verbosity normal \
                                --logger "trx;LogFileName=test-results.trx" \
                                --collect:"XPlat Code Coverage" \
                                -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover
                        '''
                    }
                }
            }
            post {
                always {
                    // Publish test results
                    script {
                        // JUnit test results
                        junit '**/test-results.trx'
                        
                        // Code coverage (if you have the plugin)
                        // publishCoverage adapters: [istanbulCoberturaAdapter('**/coverage.cobertura.xml')]
                    }
                }
            }
        }

        stage('Build Docker Image') {
            when {
                branch 'main'  // Only build images on main branch
            }
            steps {
                container('docker') {
                    script {
                        echo "=== Building Docker image ==="
                        sh """
                            docker build \
                                -t ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} \
                                -t ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:latest \
                                -f Dockerfile \
                                .
                        """
                    }
                }
            }
        }

        stage('Push Docker Image') {
            when {
                branch 'main'
            }
            steps {
                container('docker') {
                    script {
                        echo "=== Pushing to Docker registry ==="
                        sh """
                            echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin ${DOCKER_REGISTRY}
                            docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                branch 'main'
            }
            steps {
                container('dotnet') {
                    script {
                        echo "=== Deploying to Kubernetes ==="
                        // CUSTOMIZE: Add your deployment logic here
                        sh """
                            echo "Would deploy ${DOCKER_IMAGE}:${DOCKER_TAG} to Kubernetes"
                            # kubectl set image deployment/jellyfin jellyfin=${DOCKER_IMAGE}:${DOCKER_TAG} -n jellyfin
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo '=== Pipeline completed successfully! ==='
            // CUSTOMIZE: Add notifications (Slack, email, etc.)
        }
        failure {
            echo '=== Pipeline failed! ==='
            // CUSTOMIZE: Add failure notifications
        }
        always {
            // Clean up
            cleanWs()
        }
    }
}