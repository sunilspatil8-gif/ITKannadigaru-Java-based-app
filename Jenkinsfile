pipeline {
    agent any

    tools {
        jdk 'JDK21'
        maven 'maven'
    }

    environment {
        IMAGE_NAME = "manojkrishnappa/itkannadigaru-blogpost:${GIT_COMMIT}"
        AWS_REGION = "us-west-2"
        CLUSTER_NAME = "itkannadigaru-cluster"
        NAMESPACE = "microdegree"
    }

    stages {

        stage('Compile') {
            steps {
                sh '''
                    echo "===== JAVA ====="
                    java -version

                    echo "===== JAVA_HOME ====="
                    echo "$JAVA_HOME"

                    echo "===== JAVAC ====="
                    javac -version || true

                    echo "===== JAVAC PATH ====="
                    which javac || true

                    echo "===== MAVEN ====="
                    mvn -version

                    echo "===== COMPILE ====="
                    mvn compile
                '''
            }
        }
        }

        stage('packaging') {
            steps {
                sh '''
                    mvn clean package
                '''
            }
        }

        stage('docker-build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME} .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-hub-creds',
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {
                        sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'
                    }
                }
            }
        }

        stage('Push to dockerhub') {
            steps {
                sh '''
                    docker push ${IMAGE_NAME}
                '''
            }
        }

        stage('update the k8 cluster') {
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
            }
        }

        stage('Deploy to EKS cluster') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'itkannadigaru-cluster',
                    contextName: '',
                    credentialsId: 'kube',
                    namespace: 'microdegree',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://420880259B390C766ED47F436190C1B6.gr7.us-west-2.eks.amazonaws.com'
                ) {
                    sh "sed -i 's|replace|${IMAGE_NAME}|g' deployment.yml"
                    sh "kubectl apply -f deployment.yml -n ${NAMESPACE}"
                }
            }
        }

        stage('verify') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'itkannadigaru-cluster',
                    contextName: '',
                    credentialsId: 'kube',
                    namespace: 'microdegree',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://420880259B390C766ED47F436190C1B6.gr7.us-west-2.eks.amazonaws.com'
                ) {
                    sh "kubectl get pods -n ${NAMESPACE}"
                    sh "kubectl get svc -n ${NAMESPACE}"
                }
            }
        }
    }
}
