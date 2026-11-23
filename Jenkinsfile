pipeline {
    agent any

    environment {
        AWS_REGION   = "ap-southeast-2"
        CLUSTER_NAME = "k8s_cluster"
        OBS_NS       = "observability"
        APP_NS       = "microservices"
    }

    stages {
        stage('Checkout Repository') {
            steps {
                checkout scm
            }
        }

        stage('Connect to EKS Cluster') {
            steps {
                sh '''
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
                    kubectl get nodes
                '''
            }
        }

        stage('Prepare Namespaces & Helm Repos') {
            steps {
                sh '''
                    kubectl create namespace ${OBS_NS} --dry-run=client -o yaml | kubectl apply -f -
                    kubectl create namespace ${APP_NS} --dry-run=client -o yaml | kubectl apply -f -

                    PROM_URL=$(echo "aHR0cHM6Ly9wcm9tZXRoZXVzLWNvbW11bml0eS5naXRodWIuaW8vaGVsbS1jaGFydHM=" | base64 -d)
                    FLUENT_URL=$(echo "aHR0cHM6Ly9mbHVlbnQuZ2l0aHViLmlvL2hlbG0tY2hhcnRz" | base64 -d)
                    JAEGER_URL=$(echo "aHR0cHM6Ly9qYWVnZXJ0cmFjaW5nLmdpdGh1Yi5pby9oZWxtLWNoYXJ0cw==" | base64 -d)

                    helm repo add prometheus-community "$PROM_URL"
                    helm repo add fluent "$FLUENT_URL"
                    helm repo add jaegertracing "$JAEGER_URL"
                    helm repo update
                '''
            }
        }

        stage('Deploy Prometheus & Grafana') {
            steps {
                sh '''
                    helm upgrade --install kube-prometheus prometheus-community/kube-prometheus-stack \
                      --namespace ${OBS_NS} \
                      -f values/prometheus-values.yaml \
                      --wait --timeout 10m
                '''
            }
        }

        stage('Deploy Fluent Bit -> Elastic Cloud') {
            steps {
                sh '''
                    helm upgrade --install fluent-bit fluent/fluent-bit \
                      --namespace ${OBS_NS} \
                      -f values/fluent-bit-values.yaml \
                      --wait --timeout 5m
                '''
            }
        }

        stage('Deploy Jaeger Tracing') {
            steps {
                sh '''
                    helm upgrade --install jaeger jaegertracing/jaeger \
                      --namespace ${OBS_NS} \
                      -f values/jaeger-values.yaml \
                      --wait --timeout 5m

                    kubectl expose deployment jaeger -n ${OBS_NS} \
                      --name=jaeger-ui-lb \
                      --type=LoadBalancer \
                      --port=16686 --target-port=16686 \
                      --dry-run=client -o yaml | kubectl apply -f -
                '''
            }
        }

        stage('Deploy Instrumented Microservices') {
            steps {
                sh '''
                    kubectl apply -f values/apps/microservices-demo.yaml -n ${APP_NS}
                    kubectl rollout status deployment/payment-service -n ${APP_NS} --timeout=5m
                    kubectl rollout status deployment/checkout-gateway -n ${APP_NS} --timeout=5m
                '''
            }
        }

        stage('Print Access URLs') {
            steps {
                sh '''
                    echo "========================================="
                    echo "GRAFANA URL (admin / AdminPassword123!):"
                    kubectl get svc kube-prometheus-grafana -n ${OBS_NS} -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}\n'
                    echo "JAEGER UI URL:"
                    kubectl get svc jaeger-ui-lb -n ${OBS_NS} -o jsonpath='http://{.status.loadBalancer.ingress[0].hostname}:16686\n'
                    echo "========================================="
                '''
            }
        }
    }

    post {
        failure {
            sh '''
                kubectl get pods -n ${OBS_NS}
                kubectl get pods -n ${APP_NS}
            '''
        }
    }
}
