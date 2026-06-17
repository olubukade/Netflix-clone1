pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod

spec:
  serviceAccountName: jenkins

  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - /busybox/sh
    args:
    - -c
    - cat
    tty: true
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker

  - name: trivy
    image: aquasec/trivy:latest
    command:
    - cat
    tty: true

  - name: node
    image: node:18-alpine
    command:
    - cat
    tty: true

  - name: kubectl
    image: dtzar/helm-kubectl:3.14.4
    command:
    - cat
    tty: true

  volumes:
  - name: docker-config
    secret:
      secretName: regcred
      items:
      - key: .dockerconfigjson
        path: config.json
"""
        }
    }

    environment {
        IMAGE_NAME = "docker.io/olubukade95/netflix-clone"
        APP_URL = "http://10.0.0.124:32680"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Files') {
            steps {
                sh '''
                ls -la
                cat package.json
                cat Dockerfile
                cat sonar-project.properties
                cat helm/netflix-clone/values.yaml
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                container('node') {
                    sh '''
                    yarn install
                    '''
                }
            }
        }

        stage('Build React App') {
            steps {
                container('node') {
                    sh '''
                    yarn build
                    ls -la dist
                    '''
                }
            }
        }

        stage('Trivy SCA Filesystem Scan') {
            steps {
                container('trivy') {
                    sh '''
                    trivy fs \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --scanners vuln,secret \
                      .
                    '''
                }
            }
        }

        stage('SonarQube Code Scan') {
            steps {
                container('jnlp') {
                    script {
                        def scannerHome = tool 'SonarScanner'

                        withSonarQubeEnv('SonarQube') {
                            sh """
                            ${scannerHome}/bin/sonar-scanner
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                container('jnlp') {
                    timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Build and Push Image') {
            steps {
                container('kaniko') {
                    withCredentials([string(
                        credentialsId: 'tmdb-api-key',
                        variable: 'TMDB_API_KEY'
                    )]) {
                        sh '''
                        /kaniko/executor \
                          --context=$(pwd) \
                          --dockerfile=Dockerfile \
                          --build-arg TMDB_V3_API_KEY=$TMDB_API_KEY \
                          --destination=$IMAGE_NAME:$BUILD_NUMBER
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                container('trivy') {
                    sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --ignore-unfixed \
                      $IMAGE_NAME:$BUILD_NUMBER
                    '''
                }
            }
        }

        stage('Update Helm Image Tag') {
            steps {
                container('jnlp') {
                    withCredentials([usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh '''
                        sed -i "s/tag: .*/tag: $BUILD_NUMBER/" helm/netflix-clone/values.yaml

                        git config user.email "jenkins@homelab.local"
                        git config user.name "Jenkins CI"

                        git add helm/netflix-clone/values.yaml
                        git commit -m "Update Netflix clone image tag to build $BUILD_NUMBER" || echo "No changes to commit"

                        git push https://$GIT_USERNAME:$GIT_TOKEN@github.com/olubukade/Netflix-clone1.git HEAD:main
                        '''
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                container('kubectl') {
                    sh '''
                    echo "Waiting for ArgoCD/Kubernetes deployment..."
                    sleep 60

                    echo "Testing Netflix Clone..."
                    curl -f $APP_URL
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#devops-alerts',
                color: 'good',
                tokenCredentialId: 'slack-token',
                botUser: true,
                message: "SUCCESS: Netflix Clone ${env.JOB_NAME} #${env.BUILD_NUMBER} completed successfully. ${env.BUILD_URL}"
            )
        }

        failure {
            slackSend(
                channel: '#devops-alerts',
                color: 'danger',
                tokenCredentialId: 'slack-token',
                botUser: true,
                message: "FAILED: Netflix Clone ${env.JOB_NAME} #${env.BUILD_NUMBER} failed. ${env.BUILD_URL}"
            )
        }
    }
}
