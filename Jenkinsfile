pipeline {
    agent any

    stages {
        stage('Setup cluster') {
            steps {
                withAWS(credentials: 'eks_login', region: 'us-east-2') {
                    dir('eks') {
                        sh 'eksctl create cluster -f cluster.yml'
                    }
                }
            }
        }
        stage('Setup serviço DNS External') {
            steps {
                withAWS(credentials: 'eks_login', region: 'us-east-2') {
                    dir('eks') {
                        sh 'eksctl utils associate-iam-oidc-provider --region=us-east-2 --cluster=homologacao --approve'
                        sh 'eksctl create iamserviceaccount \
                            --name external-dns \
                            --namespace default \
                            --cluster homologacao \
                            --attach-policy-arn arn:aws:iam::566854824552:policy/AllowExternalDNSUpdates \
                            --approve'
                        sh 'kubectl create -f external-dns/external-dns.yml'
                    }
                }
            }
        }
        stage('Setup serviço Prometheus Operator'){
            steps {
               withAWS(credentials: 'eks_login', region: 'us-east-2') {
                    dir('eks') {
                       sh 'helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
                       sh 'helm repo update'
                       sh 'helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
                           --namespace monitoring \
                           --create-namespace \
                           --values "monitoring/values.yml"'
                    }
                }  
            }
        }
        stage('Setup serviço Ingress Nginx') {
            steps {
                withAWS(credentials: 'eks_login', region: 'us-east-2') {
                    dir('eks') {
                        sh 'helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx'
                        sh 'helm repo update'
                        sh 'helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
                            --namespace ingress-nginx \
                            --create-namespace \
                            --set controller.metrics.enabled=true \
                            --set-string controller.podAnnotations."prometheus.io/scrape"="true" \
                            --set-string controller.podAnnotations."prometheus.io/port"="10254" \
                            --set controller.service.externalTrafficPolicy=Local \
                            --values "ingress-nginx/values.yml"'
                        sh 'kubectl create -f ingress-nginx/servicemonitor.yml'
                    }
                }
            }
        }
        stage('Setup serviço Cert Manager') {
            steps {
                withAWS(credentials: 'eks_login', region: 'us-east-2') {
                    dir('eks') {
                        sh 'helm repo add jetstack https://charts.jetstack.io'
                        sh 'helm repo update'
                        sh 'helm upgrade --install cert-manager jetstack/cert-manager \
                            --namespace cert-manager \
                            --create-namespace \
                            --values "cert-manager/cert-manager-values.yml" \
                            --wait'
                        sh 'kubectl create -f cert-manager/cert-issuer.yml'
                    }
                }
            }
        }
        stage('Setup escalonamento automático do Cluster') {
            steps {
                withAWS(credentials: 'eks_login', region: 'us-east-2') {
                    dir('eks') {
                        sh 'eksctl create iamserviceaccount \
                            --cluster=homologacao \
                            --namespace=kube-system \
                            --name=cluster-autoscaler \
                            --attach-policy-arn=arn:aws:iam::566854824552:policy/ClusterAutoscalerPolicy \
                            --override-existing-serviceaccounts \
                            --approve'
                        sh 'kubectl create -f autoscaler/cluster-autoscaler-autodiscover.yml'
                    }
                }
            }
        }
        stage('Setup serviço Metrics Server') {
            steps {
                withAWS(credentials: 'eks_login', region: 'us-east-2') {
                    sh 'helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/'
                    sh 'helm repo update'
                    sh 'helm upgrade --install metrics-server metrics-server/metrics-server'
                }
            }
        }
        stage('Setup serviço Kong') {
            steps {
                withAWS(credentials: 'eks_login', region: 'us-east-2') {
                    dir('eks') {
                        sh 'kubectl create -f kong/namespace.yml'
                        sh 'kubectl create -f kong/limitrange.yml'
                        sh 'helm repo add kong https://charts.konghq.com'
                        sh 'helm repo update'
                        sh 'helm upgrade --install kong kong/kong --namespace kong --values "kong/values.yml"'
                        sh 'kubectl create -f kong/konga-deployment.yml'
                        sh 'kubectl create -f kong/konga-service.yml'
                        sh 'kubectl create -f kong/konga-ingress.yml'
                    }
                }
            }
        }
    }
}