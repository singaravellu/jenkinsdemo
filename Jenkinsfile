pipeline {

  environment {
    registry = "hub.docker.com"
    dockerImage = "nginx"
    registryCredential = "docker"
    tag = "nginxv1"
  }
  agent { label 'agent1' }
  
  stages {
    stage('git scm checkout') {
      steps {
         git branch: 'main', credentialsId: 'git', url: 'https://github.com/AnupKumar-ops/jenkinsdemo.git'
      }
    }
    
    stage('current') {
      steps{
        dir("${env.WORKSPACE}"){
          sh "pwd"
        }
      }
    }
   
    stage('Build image') {
      steps{
        sh "docker build -t 963287/myrepo:${tag} ."
      }
    }
    
    stage('push image') {
      steps {
        script {  
          docker.withRegistry( '', registryCredential ) { 
           sh "docker push 963287/myrepo:${tag}"
          } 
        }    
      }    
    }
    
    stage('start container') {
        steps {
            sh "docker run -d --name myapp -p 80:80 963287/myrepo:${tag}"
        }
    }
  }
  
}
