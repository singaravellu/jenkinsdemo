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
def upload_filepath              = "http://35.202.168.85:8082/artifactory/${VendorName}/${Product}/${Version}"
def SourceRegistry               = "hub.docker.com"
def TargetRegistry               = "docker.io/963287/myrepo"
def TargetRegistryUbuntu         = "963287/myrepo" 
def SourceDockerImage            = "nginx"
def image_version                = "latest"
def user_image_checksum          = "8e10956422503824ebb599f37c26a90fe70541942687f70bbdb744530fc9eba4"
def userInput = input(
            id: 'userInput', message: 'Enter Artifactory credentials:?',
                parameters: [

                    string(defaultValue: 'admin',
                                            description: 'username of Artifactory',
                                            name: 'username'),
                    password(defaultValue: 'None',
                                            description: 'apitoken of Artifactory',
                                            name: 'password'),
                    string(defaultValue: 'http://35.202.168.85:8082/artifactory/vendor_repo2',
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
                          for file in `ls $WORKSPACE/artifacts`
                          do
                           ARTIFACT_MD5_CHECKSUM=$(md5sum $WORKSPACE/artifacts/$file | awk '{print $1}')
                           ARTIFACT_SHA1_CHECKSUM=$(sha1sum  $WORKSPACE/artifacts/$file | awk '{ print $1 }')
                           response=$(curl -s -o /dev/null -w "%{http_code}" \
                           -u "$USER_NAME":"$ARTF_TOKEN" \
                           --connect-timeout 7 -i -X PUT  \
                           -H "X-Checksum-MD5:${ARTIFACT_MD5_CHECKSUM}" \
                           -H "X-Checksum-Sha1:${ARTIFACT_SHA1_CHECKSUM}" \
                           -T "$WORKSPACE"/artifacts/"$file" \
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
           
        stage('get Replicas') {
                  steps {
                         script {
                                def userInput3 = input(
                                  id: 'userInput3', message: 'Enter replicas:?',
                                                          parameters: [
                                                                 string(defaultValue: '2',
                                                                               description: 'replica count',
                                                                               name: 'replicasval'),
                                                          ]
                                )
                                REPLICAS       = "${userInput3}"
                         }
                  }
        }
           
        stage('Check Resource availablity in K8') {
               environment {
                    REPLICAS       = "${REPLICAS}"
               }       
               steps {
                        // check resource availablity
                                
                       sh '''
                          check() {
                             echo "Applied Requests and Limits"
                             kubectl get nodes --no-headers | awk '(NR>1)' | awk '{print $1}' | xargs -I {} sh -c 'echo {}; kubectl describe node {} | grep Allocated \
                             -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- ; echo'
                             applied_cpu=`kubectl get nodes --no-headers | awk '(NR>1)' | awk '{print $1}' | xargs -I {} sh -c 'echo {}; kubectl describe node {} | grep Allocated \
                             -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- ; echo'| grep cpu | awk -F " " '{print $2}' | awk -F "m" '{print $1}'`
                             applied_mem=`kubectl get nodes --no-headers | awk '(NR>1)' | awk '{print $1}' | xargs -I {} sh -c 'echo {}; kubectl describe node {} | grep Allocated \
                             -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- ; echo'| grep memory | awk -F " " '{print $2}' | awk -F "Mi" '{print $1}'`

                             echo "########################"

                             cpu=0
                             memory=0
                             replicas=$1
                             i=1
                             for node in $(kubectl get nodes | grep -v NAME| awk '{print $1}' | awk '(NR>1)')
                             do      
                                 sum1=$(kubectl describe nodes $node | grep cpu | awk -F " " '{print $2}'| head -1)
                                 sum2=$(kubectl describe nodes $node | grep "memory:" | awk -F " " '{print $2}'| head -1 | awk -F "K" '{print $1}')
                                 cpu=$(expr $sum1 + $cpu)
                                 memory=$(expr $sum2 + $memory)
                                 echo "--------------- $node ---------"
                                 total_allocatable_cpu=$(echo "$sum1*0.93*1000" | bc)
                                 total_allocatable_mem=$(echo "$sum2*0.75/1024" | bc)
                                 echo "Total_cpu:${total_allocatable_cpu}m"
                                 echo "Total_mem:${total_allocatable_mem}Mi"
                                 a=`echo $applied_cpu | cut --delimiter " " --fields $i`
                                 b=`echo $applied_mem | cut --delimiter " " --fields $i`
                                 allocatable_cpu=`echo "$total_allocatable_cpu-$a" | bc`
                                 allocatable_mem=`echo "$total_allocatable_mem-$b" | bc`
                                 echo "allocatable_cpu:${allocatable_cpu}m"
                                 echo "allocatable_mem:${allocatable_mem}Mi"
                                 cpu_hardlimit=$(echo "$allocatable_cpu/$replicas"| bc)
                                 mem_hardlimit=$(echo "$allocatable_mem/$replicas"| bc)
                                 echo "cpu_hardlimit:${cpu_hardlimit}m"
                                 echo "mem_hardlimit:${mem_hardlimit}Mi"
                                 i=$(( i+1 ))
                                 echo " "
                              done
                              echo "Requested resources by user"
                              cat nginx-app-chart/values.yaml | .local/bin/shyaml get-value resources
                        }
                        ssh -o StrictHostKeyChecking=no jenkins@k8-master "$(typeset -f); check $REPLICAS"
                      '''  
                  }       
         }                                                        
        
        stage('get input'){
               steps {
                      script {
                              def userInput4 = input(
                                     id: 'userInput4', message: 'Enter K8s Resource details:?',
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
                                                                         string(defaultValue: 'None',
                                                                                         description: 'application name',
                                                                                         name: 'application'),
                                                         ]
                              )
                              def userInput5 = input(
                                     id: 'userInput5', message: 'Enter pod Resource details:?',
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
                                                          ]
                              )
                              KUBE_NAMESPACE     = "${userInput4.k8_namespace?:''}"
                              LIMITS_CPU         = "${userInput4.cpu_limits?:''}"
                              LIMITS_MEMORY      = "${userInput4.memory_limits?:''}"
                              REQ_CPU            = "${userInput4.cpu_requests?:''}"
                              REQ_MEMORY         = "${userInput4.memory_requests?:''}"
                              APPLICATION        = "${userInput4.application?:''}"
                              POD_LIMITS_CPU     = "${userInput5.cpu_limits?:''}"
                              POD_LIMITS_MEMORY  = "${userInput5.memory_limits?:''}"
                              POD_REQ_CPU        = "${userInput5.cpu_requests?:''}"
                              POD_REQ_MEMORY     = "${userInput5.memory_requests?:''}"
                              
                      }
               }
        }       
         
        stage('deploy to k8') {
             environment {
                    KUBE_NAMESPACE     = "${KUBE_NAMESPACE}"
                    LIMITS_CPU         = "${LIMITS_CPU}"
                    LIMITS_MEMORY      = "${LIMITS_MEMORY}"
                    REQ_CPU            = "${REQ_CPU}"
                    REQ_MEMORY         = "${REQ_MEMORY}"
                    REPLICAS           = "${REPLICAS}"
                    APPLICATION        = "${APPLICATION}"
                    POD_LIMITS_CPU     = "${POD_LIMITS_CPU}"
                    POD_LIMITS_MEMORY  = "${POD_LIMITS_MEMORY}"
                    POD_REQ_CPU        = "${POD_REQ_CPU}"
                    POD_REQ_MEMORY     = "${POD_REQ_MEMORY}"
             }       
               steps {      
                     sh '''
                        getinputs() {
                        var=`kubectl create namespace $1`
                        if [ $? -eq 1 ]; then
                           echo "$1 namespace already created."
                           exit 1
                        else
                           kubectl create quota appquota --hard=limits.cpu=$2,limits.memory=$3,requests.cpu=$4,requests.memory=$5 -n $1
                           helm install $9 nginx-app-chart --set image.repository=$6 --set image.tag=$7 --set replicaCount=$8 --set resources.limits.cpu=$10 \
                           --set resources.limits.memory=$11 --set resources.requests.cpu=$12 --set resources.requests.memory=$13  -n $1
                        fi
                        }
                        rsync -av $WORKSPACE/nginx-app-chart jenkins@k8-master:/home/jenkins/
                        ssh -o StrictHostKeyChecking=no jenkins@k8-master "$(typeset -f); getinputs \
                        $KUBE_NAMESPACE $LIMITS_CPU $LIMITS_MEMORY \
                        $REQ_CPU $REQ_MEMORY $TARGET_REGISTRY_UBUNTU $BUILD_NUMBER $REPLICAS $APPLICATION $POD_LIMITS_CPU $POD_LIMITS_MEMORY $POD_REQ_CPU $POD_REQ_MEMORY"
                     '''
            }       
        }                 
           
   }
}


