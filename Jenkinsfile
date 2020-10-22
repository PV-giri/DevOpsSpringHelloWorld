def withDockerNetwork(Closure inner) {
  try {
    networkId = UUID.randomUUID().toString()
    sh "docker network create ${networkId}"
    inner.call(networkId)
  } finally {
    sh "docker network rm ${networkId}"
  }
}


pipeline {
    
    agent {
       node { label 'master' } 
    }
    
   environment {
        registry = "paleontolog/dev_ops"
        registryCredential = 'dockerhub'
   }
   
   stages {
         stage('Build') {
            agent {
                docker { 
                    image 'maven:3-alpine' 
                    args '-v $HOME/.m2:/root/.m2'
                }
            } 
            steps {
              git branch: 'master', credentialsId: 'github', url: 'https://github.com/Paleontolog/DevOpsSpringHelloWorld.git'
              sh 'ls'
              sh 'mvn --version'
              sh 'mvn clean package'
              sh 'ls'
              stash includes: 'target/*.jar', name: 'compiledJAR'
              stash includes: 'Dockerfile', name: 'dockerfile'
            }
        }
        
        stage('Build docker image') {
            steps {
                unstash 'dockerfile'
                unstash 'compiledJAR'
                sh 'ls'
                script {
                  def dockerHome = tool 'myDocker'
                  env.PATH = "${dockerHome}/bin:${env.PATH}"    
                   
                  def myDocker = docker.build("${registry}:${env.BUILD_ID}")
                  docker.withRegistry('', registryCredential ) {
                        myDocker.push()
                    }   
                }
            }
        }
        
        stage('Remove container') {
            steps {
                script {
                    try {
                        sh "docker rmi ${registry}:${env.BUILD_ID}"
                    } catch (exc) {
                        print("Container not found")
                    }
                }
            }
        }
        
        
        stage('Pull container') {
            steps {
                script {
                    try {
                        withDockerNetwork{ n ->
                            docker.image("${registry}:${env.BUILD_ID}").withRun("--network ${n} --name temptest") { c ->
                               docker.image('curlimages/curl').inside("""--network ${n} --entrypoint=''""") {
                                   
                                    sh '''
                                        set +x;
                                         x=0; 
                                         while [ $x -lt 100 ] && ! curl temptest:8989 --silent --output /dev/null; 
                                         do 
                                            x=$(( $x + 1 )); 
                                            sleep 1; 
                                         done
                                    '''
                                    
                                    def responce = sh(
                                        script: '''
                                            responce=$(curl --write-out '%{http_code}' \
                                                --silent --output /dev/null temptest:8989/heresy)
                                        ''', 
                                        returnStdout: true).trim()
                                    assert responce != 200
                              }
                            }
                        } 
                    } catch(ex){
                        print(ex)
                    }
                }
            }
        }
    }
    post {
        always {
              script {
                    try {
                        sh "docker rmi ${registry}:${env.BUILD_ID}"
                    } catch (exc) {
                        print(exc)
                    }
                }
        }
    }
}
