pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Image tag to deploy (use broken-v99 to test auto-rollback)')
    }

    stages {
        stage('1. Verify Cluster & Observability Stack') {
            steps {
                sh '''
                    kubectl get nodes
                    kubectl get pods -n observability
                '''
            }
        }

        stage('2. Deploy Microservice to EKS') {
            steps {
                sh """
                    kubectl set image deployment/hotrod hotrod=jaegertracing/example-hotrod:${params.IMAGE_TAG} -n demo-app
                """
            }
        }

        stage('3. Health Check & Auto-Rollback') {
            steps {
                script {
                    try {
                        sh 'kubectl rollout status deployment/hotrod -n demo-app --timeout=45s'
                        echo "SUCCESS: Rollout healthy! Monitor metrics in Grafana, logs in Kibana, and traces in Jaeger."
                    } catch (err) {
                        echo "FAILURE DETECTED: Pod failed to start (ImagePullBackOff / CrashLoopBackOff). Triggering rollback..."
                        sh 'kubectl rollout undo deployment/hotrod -n demo-app'
                        error("Deployment failed and was automatically rolled back to the previous stable version.")
                    }
                }
            }
        }
    }
}
