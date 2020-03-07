pipeline {
  agent {
    kubernetes {
      label 'jenkins-slave'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:18.09
    command: ['cat']
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock  
  - name: tools
    image: argoproj/argo-cd-ci-builder:v0.13.1
    command:
    - cat
    tty: true 
  - name: kubectl
    image: devkube/kubectl-docker
    command:
    - version
    tty: true
    volumeMounts:
    - name: kubeconfig
      mountPath: /.kube/config 
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
  - name: kubeconfig
    hostPath:
      path: /home/ubuntu/.kube/config
"""
    }
  }
  stages {

    stage('Build & Push') {
      steps {
        container('docker') {
          // Build new image
            sh "docker build -t 10.10.10.18:5000/test:${env.GIT_COMMIT} ."
          // Publish new image
            sh "docker push 10.10.10.18:5000/test:${env.GIT_COMMIT}"
        }
      }
    }

    stage('Promote to Staging') {
      environment {
        GIT_CREDS = credentials('git')
      }
      steps {
        container('tools') {
          sh "git clone https://$GIT_CREDS_USR:$GIT_CREDS_PSW@github.com/IAMDEH/test-k8s-deploy.git"
          sh "git config --global user.email 'ci@ci.com'"
          dir("test-k8s-deploy") {
            sh "cd ./kustomize/e2e && kustomize edit set image 10.10.10.18:5000/test:${env.GIT_COMMIT}"
            sh "git commit -am 'Publish new version' && git push || echo 'no changes'"
          }
        }
      }
    }

    stage('Deploy to Staging'){
      steps {
        container('kubectl'){
          dir("test-k8s-deploy"){
            sh "kubectl -n test-e2e apply -k ./kustomize/e2e"
          }
        }
      }
    }
/*
    stage('Promote to Prod') {
      steps {
        input message:'Approve deployment?'
        container('tools') {
          dir("test-jenkins-deploy") {
            sh "cd ./prod && kustomize edit set image 10.10.10.16:5000/test:${env.GIT_COMMIT}"
            sh "git commit -am 'Publish new version' && git push || echo 'no changes'"
          }
        }
      }
    }
*/
  }
}