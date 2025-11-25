pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'ahmedtaha141414'
        APP_IMAGE = 'k8s-app-repo'
        DB_IMAGE = 'k8s-db-repo'
        TAG = 'latest'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/vhmedtaha/Production-Ready-DevOps-CI-CD-for-Modern-Java-Apps.git'
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dir('Docker-files/app') {
                        sh "docker build -t ${DOCKERHUB_USER}/${APP_IMAGE}:${TAG} ."
                    }
                }
            }
        }

        stage('Build DB Image') {
            steps {
                script {
                    dir('Docker-files/db') {
                        sh "docker build -t ${DOCKERHUB_USER}/${DB_IMAGE}:${TAG} ."
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push ${DOCKERHUB_USER}/${APP_IMAGE}:${TAG}
                        docker push ${DOCKERHUB_USER}/${DB_IMAGE}:${TAG}
                        docker logout
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    # Apply Kubernetes YAML files without verifying TLS
                    kubectl --insecure-skip-tls-verify=true apply -f app-secret.yml
                    kubectl --insecure-skip-tls-verify=true apply -f db-CIP.yml
                    kubectl --insecure-skip-tls-verify=true apply -f mc-CIP.yml
                    kubectl --insecure-skip-tls-verify=true apply -f mcdep.yml
                    kubectl --insecure-skip-tls-verify=true apply -f rmq-CIP-service.yml
                    kubectl --insecure-skip-tls-verify=true apply -f rmq-dep.yml
                    kubectl --insecure-skip-tls-verify=true apply -f vproapp-service.yml
                    kubectl --insecure-skip-tls-verify=true apply -f vproappdep.yml
                    kubectl --insecure-skip-tls-verify=true apply -f vprodbdep.yml
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Build and deployment completed successfully."
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
