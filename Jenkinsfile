pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'shrikantghyal/my-app'
        AWS_REGION = 'ap-south-1'
        CLUSTER_NAME = 'ecommerce-cluster'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build(DOCKER_IMAGE)
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-creds') {
                        docker.image(DOCKER_IMAGE).push('latest')
                    }
                }
            }
        }

        stage('Update Kubeconfig for EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS-cred']]) {
                    sh '''
                        aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME --kubeconfig /var/lib/jenkins/.kube/config
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS-cred']]) {
                    sh '''
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        kubectl apply -f deployment.yaml --validate=false
                        kubectl apply -f service.yaml --validate=false
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment Successful!'
        }
        failure {
            echo '❌ Deployment Failed!'
        }
    }
}
