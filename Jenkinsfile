@Library('github.com/AnupKumar-ops/poc')_

def SourceRepo              = "https://github.com/AnupKumar-ops/jenkinsdemo"
def SourceCredentials       = "github" 
 
pipeline {
    
    environment {
        DockerImage = ""
    }
    
   
  agent any
 
  triggers { 
     githubPush()
  }

  stages {
     stage('SCM checkout') {
           steps {
             script {
              git credentialsId: "${SourceCredentials}", url: "${SourceRepo}"
             }
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
                def server = Artifactory.newServer url: data.ArtifactoryUrl(), credentialsId: data.ArtifactoryCredentials()
                def uploadSpec = """{
                                      "files": [
                                          {
                                             "pattern": "${WORKSPACE}/*.war",
                                             "target": "data.VendorName()/data.Product()/data.Version()/"
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
               dockerImage = docker.build data.Registry() + ":$BUILD_NUMBER"
               
             }
         }
     }
     
     stage('Push Image') {
         steps {
             script {
                 docker.withRegistry( '', data.RegistryCredential() ) {
                     dockerImage.push()
                 } 
             }
         }           
     }

  }
 
   post { 
        failure { 
            echo 'Build Pipeline NOK'
        }
    }
}
