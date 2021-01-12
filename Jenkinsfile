pipeline {

  environment {
    vendor_registry = "hub.docker.com"
    vendor_dockerImage = "nginx"
    vendor_registryCredential = "docker"
    image_versiontag = "latest"
  }
  agent { label 'agent1' }
  
  stages {
    
    stage('git scm checkout') {
      steps {
         git branch: 'main', credentialsId: 'git', url: 'https://github.com/AnupKumar-ops/jenkinsdemo.git'
      }
   }
    stage('Pull Image') {
        steps {
            script { 
                docker.withRegistry( '', vendor_registryCredential ) {
                }
                sh "/usr/bin/docker pull $vendor_dockerImage:$image_versiontag"
            }        
        }
    }
  }
}
