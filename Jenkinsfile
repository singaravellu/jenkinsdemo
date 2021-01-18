def artifactory_url = "http://34.69.242.224:8082/artifactory"
def credentials     = "jfrogid"
def vendor_repo     = "cisco_vnf_v1.0"
def vendor_docker_repo = "myrepo"
def app = "tomcat_app"

 
pipeline {
    
   environment {
    registry = "hub.docker.com"
    vendor_registryCredential = "docker"
   }
   
  agent any

  stages {
     stage('SCM checkout') {
           steps {
              git branch: 'main', credentialsId: 'github', url: 'https://github.com/AnupKumar-ops/jenkinsdemo.git'
           }
     }

     stage('Upload to JFrog') { 
           steps {
             script {
                def server = Artifactory.newServer url: "${artifactory_url}", credentialsId: "${credentials}"
                def uploadSpec = """{
                                      "files": [
                                          {
                                             "pattern": "${WORKSPACE}/*.war",
                                             "target": "${vendor_repo}/"
                                          }
                                       ]
                                  }"""
                server.upload spec: uploadSpec
              }
            }
     }
     stage('build image') {
         steps {
             sh "docker build -t 963287/${vendor_docker_repo}:$BUILD_NUMBER ."
         }
     }
     
     stage('push image') {
         steps {
             script {
                 docker.withRegistry( '', vendor_registryCredential ) {
                     sh "docker push  963287/myrepo:$BUILD_NUMBER"
                 } 
             }
         }           
     }

      stage('start app') {
         steps {
            script {
                 docker.withRegistry( '', vendor_registryCredential ) {
                     sh "docker run -d --name ${app} -p 80:8080  963287/${vendor_docker_repo}:$BUILD_NUMBER"
                 } 
            }
         }              
      }
  }
  
}
