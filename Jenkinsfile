pipeline {
    agent any

    tools {
        jdk 'JDK21'
        maven 'maven'
    }

   environment {
    IMAGE_NAME = "sunilpatil08/itkannadigaru-blogpost:${GIT_COMMIT}"
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

        stage('Packaging') {
            steps {
                sh '''
                    mvn clean package
                '''
            }
        }

        stage('Docker Build') {
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
                        sh '''
                            echo "$DOCKER_PASSWORD" | docker login \
                                -u "$DOCKER_USERNAME" \
                                --password-stdin
                        '''
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                    docker push ${IMAGE_NAME}
                '''
            }
        }

        stage('Update EKS Cluster') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${CLUSTER_NAME}
                '''
            }
        }

        stage('Deploy to EKS') {
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
                    sh '''
                        sed -i "s|replace|${IMAGE_NAME}|g" deployment.yml
                        kubectl apply -f deployment.yml -n ${NAMESPACE}
                    '''
                }
            }
        }

        stage('Verify') {
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
                    sh '''
                        kubectl get pods -n ${NAMESPACE}
                        kubectl get svc -n ${NAMESPACE}
                    '''
                }
            }
        }
    }
}
