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
      image: argoproj/argocd:latest
      command: ["sh", "-c", "cat"]
      tty: true
"""
    }
  }

  environment {
    ARGOCD_SERVER = 'host.docker.internal:8080'   // adjust later if needed
    ARGOCD_USER   = 'admin'
    ARGOCD_PASS   = credentials('argocd-admin-password')
  }

  triggers {
    // Poll GitHub every 1 minute for changes
    pollSCM('* * * * *')
  }

  stages {
    stage('Check argocd CLI') {
      steps {
        container('argocd-cli') {
          sh '''
            echo "🔎 Checking argocd client..."
            which argocd || echo "argocd not in PATH"
            argocd version --client || echo "argocd version failed"
          '''
        }
      }
    }

    stage('Login to ArgoCD') {
      steps {
        container('argocd-cli') {
          sh '''
            echo "🔐 Logging into ArgoCD at ${ARGOCD_SERVER}..."
            argocd login ${ARGOCD_SERVER} \
              --username ${ARGOCD_USER} \
              --password ${ARGOCD_PASS} \
              --insecure \
              --grpc-web
          '''
        }
      }
    }

    stage('Sync Applications') {
      steps {
        container('argocd-cli') {
          sh '''
            echo "🔁 Syncing applications..."
            argocd app sync payments --grpc-web
            argocd app sync booking  --grpc-web
            argocd app sync search   --grpc-web

            echo "⏱ Waiting for apps to become healthy..."
            argocd app wait payments --health --timeout 300 --grpc-web
            argocd app wait booking  --health --timeout 300 --grpc-web
            argocd app wait search   --health --timeout 300 --grpc-web
          '''
        }
      }
    }
  }
}
