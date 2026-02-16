pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = "rrgowd/ecommerce:${BUILD_NUMBER}"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/rrgowd/Ecommerce-App-Kastro.git'
            }
        }

        stage('Maven Clean Compile') {
            steps {
                sh "mvn clean compile"
            }
        }

        stage('Maven Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=ECommerce \
                        -Dsonar.projectKey=ECommerce \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('Maven Package') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: '2b9ca1df-55d8-4539-a995-2df4c6ea61e1') {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Docker Push') {
            steps {
                withDockerRegistry([credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/']) {
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh """
                    docker stop ecommerce-container || true
                    docker rm ecommerce-container || true
                    docker run -d --name ecommerce-container -p 8083:8080 ${DOCKER_IMAGE}
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    export KUBECONFIG=/var/snap/jenkins/4992/workspace/k8s/config

                    kubectl get nodes

                    # Apply deployment yaml from your repo
                    kubectl apply -f deployment-service.yaml

                    kubectl get pods -A
                """
            }
        }

    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                        <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${env.BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: "rakeshgowd01@gmail.com",
                    from: "rakeshgowd01@gmail.com",
                    replyTo: "rakeshgowd01@gmail.com",
                    mimeType: "text/html"
                )
            }
        }
    }
}
