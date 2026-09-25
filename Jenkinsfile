pipeline {
    agent any

    tools {
        jdk 'JDK21'
        maven 'maven'
    }

    environment {
        IMAGE_NAME   = "sunilpatil08/itkannadigaru-blogpost:${GIT_COMMIT}"
        AWS_REGION   = "us-west-2"
        CLUSTER_NAME = "itkannadigaru-cluster"
        NAMESPACE    = "microdegree"
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

        // ---------------------------------------------------------------
        // IMPORTANT: `aws eks update-kubeconfig` does NOT embed a static
        // token. It writes an `exec:`-based kubeconfig that shells out to
        // `aws eks get-token` on every `kubectl` call, using whatever AWS
        // credentials are in the environment *at that moment*.
        //
        // The original pipeline scoped AWS_ACCESS_KEY_ID/SECRET only to
        // the "Update EKS Cluster" stage, then tried to deploy in a later
        // stage wrapped in a DIFFERENT `withKubeConfig(credentialsId:
        // 'kube', serverUrl: '<no scheme>')` block. That overwrote the
        // good kubeconfig with an unauthenticated/malformed one, which is
        // exactly what produced:
        //   "the server has asked for the client to provide credentials"
        //
        // Fix: keep AWS credentials in scope for every stage that runs
        // kubectl, and drop the conflicting withKubeConfig/'kube' cred
        // entirely — the kubeconfig from update-kubeconfig is sufficient.
        // ---------------------------------------------------------------

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        echo "===== AWS IDENTITY ====="
                        aws sts get-caller-identity

                        echo "===== UPDATE KUBECONFIG ====="
                        aws eks update-kubeconfig \
                            --region ${AWS_REGION} \
                            --name ${CLUSTER_NAME}

                        echo "===== DEPLOYING TO EKS ====="
                        sed -i "s|replace|${IMAGE_NAME}|g" deployment.yml
                        kubectl apply -f deployment.yml -n ${NAMESPACE}
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        echo "===== PODS ====="
                        kubectl get pods -n ${NAMESPACE}

                        echo "===== SERVICES ====="
                        kubectl get svc -n ${NAMESPACE}
                    '''
                }
            }
        }
    }
}
