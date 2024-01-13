pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'nodejs16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = "Reactjs-Blog"
        RELEASE = "1.0.0"
        DOCKER_USER = "nikhil999999"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from git') {
            steps {
                git branch: 'main', url: 'https://github.com/Devnikops/Reactjs-Blog.git'
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Blog-CICD \
                    -Dsonar.projectKey=Blog-CICD'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SonarQube-Token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage('Build and Push Docker image') {
            steps {
                script {
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                        
                    }
                }
            }
        }

        stage ('Trivy Image Scan') {
            steps {
                script {
                    sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image nikhil999999/Reactjs-Blog:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table > trivyimage.txt')
                }
            }
        }

        stage ('Clean Artificates') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }

        stage ('Trigger Reactjs-Blog CD Job') {
            steps {
                script {
                    sh "curl -v -k --user devopsuser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-13-200-154-111.ap-south-1.compute.amazonaws.com:8080/job/Blog-App-CD/buildWithParameters?token=gitops-token'"
                }
            }
        }

    }

    post {
        always {
            emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
                to: 'devopswithnik@gmail.com',                              
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }

}
