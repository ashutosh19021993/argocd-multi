pipeline {
    agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: argo-sync-agent
spec:
  containers:
    - name: argocd-cli
      image: argoproj/argocd:v3.1.9
      command: ["sh", "-c", "cat"]
      tty: true
"""
    }
  }

    environment {
        ARGOCD_SERVER = 'host.docker.internal:8080'
        ARGOCD_USER   = 'admin'
        ARGOCD_PASS   = credentials('argocd-admin-password')
    }

    triggers {
        // Poll GitHub every 1 minute for changes on main
        pollSCM('* * * * *')
    }

    stages {
        stage('Login to ArgoCD') {
            steps {
                sh '''
                  argocd login ${ARGOCD_SERVER} \
                    --username ${ARGOCD_USER} \
                    --password ${ARGOCD_PASS} \
                    --insecure \
                    --grpc-web
                '''
            }
        }

        stage('Sync Applications') {
            steps {
                sh '''
                  argocd app sync payments  --grpc-web
                  argocd app sync booking   --grpc-web
                  argocd app sync search    --grpc-web

                  argocd app wait payments --health --timeout 300 --grpc-web
                  argocd app wait booking  --health --timeout 300 --grpc-web
                  argocd app wait search   --health --timeout 300 --grpc-web
                '''
            }
        }
    }
}
