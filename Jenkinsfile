pipeline {

  environment {
    vendor_registry = "hub.docker.com"
    vendor_dockerImage = "nginx"
    vendor_registryCredential = "docker"
    image_versiontag = "latest"
  }
  agent { label: 'agent1' }
  
  stages {
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
