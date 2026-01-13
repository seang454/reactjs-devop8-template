@Library('devop_itp_share_library@master') _

pipeline {
    agent any

    environment {
        REPO_NAME = 'seang454'
        IMAGE_NAME = 'jenkins-itp-reactjs'
        TAG = 'latest'
    }

    stages {

        stage('Clone Code') {
            steps {
                git 'https://github.com/seang454/reactjs-devop8-template.git'
            }
        }

        stage('Install Dependencies & Test') {
            steps {
                sh '''
                npm install
                npm test -- --coverage || true
                '''
            }
        }

    stage('SonarQube Scan') {
        steps {
            withSonarQubeEnv('sonarqube-scanner') {
                script {
                    def scannerHome = tool 'sonarqube-scanner'
                    sh """
                        ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=jobfinder-frontend \
                            -Dsonar.projectName=JobFinder-Frontend \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=node_modules/**,.next/**,dist/** \
                            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                    """
                }
            }
        }
    }


        // stage('Check Quality Gate') {
        //     steps {
        //         timeout(time: 5, unit: 'MINUTES') {
        //             script {
        //                 def qg = waitForQualityGate()
        //                 if (qg.status != 'OK') {
        //                     error "Quality Gate failed: ${qg.status}"
        //                 }
        //             }
        //         }
        //     }
        // }
    stage('Prepare Dockerfile') {
        steps {
            script {
                if (!fileExists('Dockerfile')) {
                    echo 'Dockerfile not found in project. Using Dockerfile from shared library.'

                    def dockerfileContent = libraryResource 'reactjs/dev.Dockerfile'
                    writeFile file: 'Dockerfile', text: dockerfileContent
                } else {
                    echo 'Dockerfile found in project. Using project Dockerfile.'
                }
            }
        }
    }

        stage('Build Image') {
            steps {
                sh 'docker build -t ${REPO_NAME}/${IMAGE_NAME}:${TAG} .'
            }
        }

        stage('Ensure Docker Hub Repo Exists') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB-CREDENTIAL',
                    usernameVariable: 'DH_USERNAME',
                    passwordVariable: 'DH_PASSWORD'
                )]) {
                    sh '''
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" -u "$DH_USERNAME:$DH_PASSWORD" \
                        https://hub.docker.com/v2/repositories/$REPO_NAME/$IMAGE_NAME/)

                    if [ "$STATUS" -eq 404 ]; then
                        curl -s -u "$DH_USERNAME:$DH_PASSWORD" -X POST \
                            https://hub.docker.com/v2/repositories/ \
                            -H "Content-Type: application/json" \
                            -d "{\"name\":\"$IMAGE_NAME\",\"is_private\":false}"
                    fi
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB-CREDENTIAL',
                    usernameVariable: 'DH_USERNAME',
                    passwordVariable: 'DH_PASSWORD'
                )]) {
                    sh '''
                    echo "$DH_PASSWORD" | docker login -u "$DH_USERNAME" --password-stdin
                    docker push ${REPO_NAME}/${IMAGE_NAME}:${TAG}
                    '''
                }
            }
        }

        stage('Run Service') {
            steps {
                sh '''
                docker stop react-app || true
                docker rm react-app || true
                docker run -dp 3001:8080 --name react-app ${REPO_NAME}/${IMAGE_NAME}:${TAG}
                '''
            }
        }
    }
}
