def withSecretEnv(List<Map> varAndPasswordList, Closure closure) {
       wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: varAndPasswordList]) {
            withEnv(varAndPasswordList.collect { "${it.var}=${it.password}" }) {
               closure()
            }
       }
}

def SourceGitRepo                = "https://github.com/AnupKumar-ops/jenkinsdemo"
def VendorName                   = "Cisco"
def Product                      = "VNF"
def Version                      = "4.0"
def upload_filepath              = "http://104.197.48.153:8082/artifactory/${VendorName}/${Product}/${Version}"
def SourceRegistry               = "hub.docker.com"
def TargetRegistry               = "docker.io/963287/myrepo"
def TargetRegistryUbuntu         = "963287/myrepo" 
def SourceDockerImage            = "nginx"
def image_version                = "latest"
def user_image_checksum          = "10b8cc432d56da8b61b070f4c7d2543a9ed17c2b23010b43af434fd40e2ca4aa"
def userInput = input(
            id: 'userInput', message: 'Enter credentials:?',
                parameters: [

                    string(defaultValue: 'admin',
                                            description: 'username of Artifactory',
                                            name: 'username'),
                    password(defaultValue: 'None',
                                            description: 'apitoken of Artifactory',
                                            name: 'password'),
                    string(defaultValue: 'http://104.197.48.153:8082/artifactory/vendor_repo2',
                                            description: 'download path of files',
                                            name: 'Source_Repository'),
                    string(defaultValue: 'helloworld.war SampleAebApp.war',
                                            description: 'Files to be downloaded',
                                            name: 'Source_file_lists')
                ]
)                
def userInput2 = input(
            id: 'userInput2', message: 'Enter Registry credentials:?',
                parameters: [

                   
                    string(defaultValue: '963287',
                                            description: 'user of SourceRegistry',
                                            name: 'Source_registry_user'),                        
                    password(defaultValue: 'None',
                                            description: 'accesstoken of SourceRegistry',
                                            name: 'Source_registry_password'), 
                ]
)
def userInput3 = input(
            id: 'userInput2', message: 'Enter K8s Resource details:?',
                parameters: [
        
                        string(defaultValue: '1',
                                            description: 'CPU LIMITS',
                                            name: 'cpu_limits'),                        
                        string(defaultValue: '1Gi',
                                            description: 'MEMORY LIMITS',
                                            name: 'memory_limits'), 
                        string(defaultValue: '500m',
                                            description: 'CPU REQUESTS',
                                            name: 'cpu_requests'),
                        string(defaultValue: '500Mi',
                                            description: 'MEMORY REQUESTS',
                                            name: 'memory_requests'),
                        string(defaultValue: 'app-nginx',
                                            description: 'k8 namespace',
                                            name: 'k8_namespace'),
                        string(defaultValue: '2',
                                            description: 'replica count',
                                            name: 'replicas'),                        
                ]
)                            
pipeline {
    
    environment {
        SOURCE_GIT_REPO    = "${SourceGitRepo}"
        VENDOR_DOCKERIMAGE = "${SourceDockerImage}"
        VERSION_IMAGE      = "${image_version}"
        SOURCE_REGISTRY    = "${SourceRegistry}"
        TARGET_REGISTRY    = "${TargetRegistry}"
        TARGET_REGISTRY_UBUNTU = "${TargetRegistryUbuntu}"
        USER_IMAGE_CHECKSUM = "${user_image_checksum}"
        VENDOR_NAME        = "${VendorName}"
        PRODUCT            = "${Product}"
        VERSION            = "${Version}"
        UPLOAD_FILEPATH    = "${upload_filepath}"
        ARTFT_USER         = "${userInput.username?:''}"
        ARTFT_TOKEN        = "${userInput.password?:''}"
        SOURCE_PATH        = "${userInput.Source_Repository?:''}"
        FILE_LISTS         = "${userInput.Source_file_lists?:''}"
        SOURCE_REG_USER    = "${userInput2.Source_registry_user?:''}"
        SOURCE_REG_TOKEN   = "${userInput2.Source_registry_password?:''}"
        KUBE_NAMESPACE          = "${userInput3.k8_namespace?:''}"
        LIMITS_CPU           = "${userInput3.cpu_limits?:''}"
        LIMITS_MEMORY        = "${userInput3.memory_limits?:''}"
        REQ_CPU           = "${userInput3.cpu_requests?:''}"
        REQ_MEMORY           = "${userInput3.memory_requests?:''}"
        REPLICAS           = "${userInput3.replicas?:''}"
    }
    
    agent any
       
    triggers { 
     githubPush()
  }   
 
    stages {
        
        stage('SCM Checkout') {
            steps {
               git changelog: false, poll: false, url: "$SOURCE_GIT_REPO"
            }
        } 
        
        stage('download vendor artifacts') {
            steps {
                sh '''
                  get() {
                      rm -rf $WORKSPACE/artifacts/*
                      cd $WORKSPACE/artifacts
                      for file in $FILE_LISTS
                      do
                        curl -O "$SOURCE_PATH/$file"
                      done
                  }
                  if [ -d "$WORKSPACE/artifacts" ]; then
                        get 
                  else
                       mkdir $WORKSPACE/artifacts
                       get
                  fi 
                '''
            }
        }
        
        stage('Upload to Artifactory') {
            
            steps {
                withSecretEnv([[var: 'USER_NAME', password: "${env.ARTFT_USER}"], [var: 'ARTF_TOKEN', password: "${env.ARTFT_TOKEN}"]]) {
                sh '''
                          for file in `ls $WORKSPACE`
                          do
                           ARTIFACT_MD5_CHECKSUM=$(md5sum $file | awk '{print $1}')
                           ARTIFACT_SHA1_CHECKSUM=$(sha1sum  $file | awk '{ print $1 }')
                           response=$(curl -s -o /dev/null -w "%{http_code}" \
                           -u "$USER_NAME":"$ARTF_TOKEN" \
                           --connect-timeout 7 -i -X PUT  \
                           -H "X-Checksum-MD5:${ARTIFACT_MD5_CHECKSUM}" \
                           -H "X-Checksum-Sha1:${ARTIFACT_SHA1_CHECKSUM}" \
                           -T "$WORKSPACE"/"$file" \
                           "$UPLOAD_FILEPATH/$file")
                            if [[ $response -eq 200 ]] ||  [[ $response -eq 201  ]]; then
                              echo "Checksum validation passed. Files Uploaded"
                            elif [[ $response -eq 409 ]]; then
                              echo "Conflicting Error. Checksum validation failed. Files upload denied"
                              exit 1
                            elif [[ $response -eq 401 ]]; then
                              echo "Unauthorized Error"
                              exit 1  
                            elif [[ $response -eq 403 ]]; then
                              echo "Forbidden Error"
                              exit 1
                            else
                              exit 1
                            fi
                          done
                '''
                }
            }
            
        }
        stage('Docker Image pull and push') {
            steps {
                withSecretEnv([[var: 'REG_USER', password: "${env.SOURCE_REG_USER}"], [var: 'REG_TOKEN', password: "${env.SOURCE_REG_TOKEN}"]]) {
                    sh '''
                        docker pull $VENDOR_DOCKERIMAGE:$VERSION_IMAGE
                        userchecksum_sha256=$(docker images --digests | grep $VENDOR_DOCKERIMAGE | grep $VERSION_IMAGE |awk '{print $3}'| awk -F ":" '{print $2}')
                        if [ $USER_IMAGE_CHECKSUM == $userchecksum_sha256 ]; then
                            sudo chmod 666 /var/run/docker.sock
                            docker tag $VENDOR_DOCKERIMAGE:$VERSION_IMAGE $TARGET_REGISTRY:$BUILD_NUMBER 
                            docker login -u "$REG_USER" -p "$REG_TOKEN" docker.io
                            docker push $TARGET_REGISTRY:$BUILD_NUMBER
                            exit 0
                        else
                            echo "Checksum validation failed. Image push denied"
                            exit 1
                        fi
                    '''
                }    
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                     getinputs() {
                      var=`kubectl create namespace $1`
                      if [ $? -eq 1 ]; then
                         echo "$1 namespace already created. Choose other name"
                         exit 1
                      else
                        kubectl create quota appquota --hard=limits.cpu=$2,limits.memory=$3,requests.cpu=$4,requests.memory=$5 -n $1
                        helm install app1 nginx-app-chart --set image.repository=$6 --set image.tag=$7 --set replicaCount=$8 -n $1
                      fi
                    }
                    rsync -av $WORKSPACE/nginx-app-chart jenkins@k8-master:/home/jenkins/
                    ssh -o StrictHostKeyChecking=no jenkins@k8-master "$(typeset -f); getinputs \
                    $KUBE_NAMESPACE $LIMITS_CPU $LIMITS_MEMORY \
                    $REQ_CPU $REQ_MEMORY $TARGET_REGISTRY_UBUNTU $BUILD_NUMBER $REPLICAS"
                '''
            } 
       }        
        
    }
}


