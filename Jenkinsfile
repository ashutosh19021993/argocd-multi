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
      image: alpine:3.20
      command: ["/bin/sh", "-c", "cat"]
      tty: true
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

  volumes:
    - name: workspace-volume
      emptyDir: {}
"""
    }
  }

  environment {
    // Argo CD is running in the same cluster as Jenkins
    // and exposed by the argocd-server Service in the argocd namespace.
    ARGOCD_SERVER = 'argocd-server.argocd.svc.cluster.local:443'
    ARGOCD_USER   = 'admin'
    ARGOCD_PASS   = credentials('argocd-admin-password')
  }

  // Optional: this makes Jenkins poll the Git repo every minute
  triggers {
    pollSCM('* * * * *')
  }

  stages {
    stage('Install & Check argocd CLI') {
      steps {
        container('argocd-cli') {
          sh '''
            echo "🔧 Installing argocd CLI in this agent pod..."

            apk add --no-cache curl

            curl -sSL -o /usr/local/bin/argocd \
              https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

            chmod +x /usr/local/bin/argocd

            echo "✅ argocd client version:"
            argocd version --client
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

            echo "✅ Login OK"
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

            echo "✅ All apps synced & healthy."
          '''
        }
      }
    }
  }
}
