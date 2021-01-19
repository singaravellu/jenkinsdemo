def VendorName              = "Telco"
def Product                 = "WarFiles"
def Version                 = "vnf_v1.0"
def ArtifactoryUrl          = "http://130.211.238.246:8082/artifactory"
def ArtifactoryCredentials = "jfrogid"
def App                     = "app2"
def SourceRepo              = "https://github.com/AnupKumar-ops/jenkinsdemo"
def SourceCredentials       = "github"       
def Registry                = "docker.io/963287/myrepo"
def RegistryCredential      = "docker"

 
pipeline {
    
    environment {
        DockerImage = ""
    }
    
   
  agent any

  stages {
     stage('SCM checkout') {
           steps {
              git credentialsId: "${SourceCredentials}", url: "${SourceRepo}"
           }
     }
   
     stage('checksum') {
         steps {
             script {
                 fingerprint '**/*.war'
             }
         }
     }
     stage('Upload to JFrog') { 
           steps {
             script {
                def server = Artifactory.newServer url: "${ArtifactoryUrl}", credentialsId: "${ArtifactoryCredentials}"
                def uploadSpec = """{
                                      "files": [
                                          {
                                             "pattern": "${WORKSPACE}/*.war",
                                             "target": "${VendorName}/${Product}/${Version}/"
                                          }
                                       ]
                                  }"""
                server.upload spec: uploadSpec
              }
            }
     }
     stage('Build image') {
         steps {
             script {
               dockerImage = docker.build "${Registry}" + ":$BUILD_NUMBER"
             }
         }
     }
     
     stage('Push Image') {
         steps {
             script {
                 docker.withRegistry( '', "${RegistryCredential}" ) {
                     dockerImage.push()
                 } 
             }
         }           
     }

  }
  
}
