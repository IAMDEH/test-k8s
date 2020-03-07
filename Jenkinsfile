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
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: kubeconf
      mountPath: /home/ubuntu/.minikube 
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
  - name: kubeconf
    hostPath:
      path: /home/ubuntu/.minikube
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

        container('kubectl'){
          withEnv(['PATH+EXTRA=/usr/sbin:/usr/bin:/sbin:/bin']) {
            sh("kubectl config --kubeconfig=config set-cluster minikube --server=https://10.10.10.18:8443 --certificate-authority=/home/ubuntu/.minikube/ca.crt")
            sh("kubectl config --kubeconfig=config set-credentials minikube --client-certificate=/home/ubuntu/.minikube/client.crt --client-key=/home/ubuntu/.minikube/client.key")
            sh("kubectl config --kubeconfig=config set-context minikube --cluster=minikube --namespace=test-e2e --user=minikube")
            sh("kubectl config use-context minikube") 
            sh("kubectl config view") 
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
