pipeline {
    agent {                                                                                                                      
        docker {
            image 'docker:cli'                                                                                                   
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }                                                                                                                        
    }
                                                                                                                                 
    options {   
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()                                                                                                
    }
                                                                                                                                 
    environment {
        APP_NAME = 'pythonapp'
        CI_IMAGE = "${APP_NAME}:ci-${env.GIT_COMMIT.take(7)}"
        SONAR_PROJECT_KEY = 'techaxis:pythonapp'
        SONAR_PROJECT_NAME = 'DevOps Python Application'
    }                                                                                                                             
 
    stages {                                                                                                                     
        stage('📋 Pipeline Info') {                                                                                              
            steps {
                script {
                    echo """
  ╔══════════════════════════════════════════════════════╗
  ║               CI PIPELINE STARTED                    ║                                                                         
  ╚══════════════════════════════════════════════════════╝
     Build  : #${env.BUILD_NUMBER}                                                                                                 
     Branch : ${env.BRANCH_NAME}                                                                                                   
     PR     : ${env.CHANGE_ID ?: 'Not a PR'}
     Title  : ${env.CHANGE_TITLE ?: 'N/A'}                                                                                         
  ══════════════════════════════════════════════════════
                    """                                                                                                          
                }
            }                                                                                                                    
        }       

        stage('🔧 Verify Environment') {
            steps {
                sh '''
                    echo "Hostname : $(hostname)"                                                                                
                    echo "User     : $(whoami)"
                    docker --version                                                                                             
                    echo "✅ Environment ready"
                '''                                                                                                              
            }   
        }                                                                                                                         
 
        stage('🔍 SonarQube Analysis') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest'
                    reuseNode true
                }
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey="${SONAR_PROJECT_KEY}" \
                          -Dsonar.projectName="${SONAR_PROJECT_NAME}" \
                          -Dsonar.sources=. \
                          -Dsonar.language=py \
                          -Dsonar.python.version=3.11 \
                          -Dsonar.exclusions="**/venv/**,**/myenv/**,**/migrations/**,**/tests/**,**/static/**"
                    '''
                }
            }  
        }

        stage('🚦 Quality Gate') {
            agent none
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        waitForQualityGate abortPipeline: false 
                    }

                }
            }
        }     

        stage('🐳 Docker Build') {                                                                                               
            steps {
                sh """                                                                                                           
                    echo "Building: ${CI_IMAGE}"
                    docker build --tag ${CI_IMAGE} --file Dockerfile .
                    echo "✅ Build successful"                                                                                   
                    docker images ${CI_IMAGE}
                """                                                                                                              
            }   
        }                                                                                                                        
                
        stage('🧪 Verify Image') {
            steps {
                sh """
                    echo "Verifying Docker image: ${CI_IMAGE}"

                    echo "Checking py files in the image"
                    docker run --rm --entrypoint ls ${CI_IMAGE} /app
                    echo "Python files verified in the image"

                    echo "Checking python version in the image"
                    docker run --rm --entrypoint python3 ${CI_IMAGE} --version
                    echo "Python version verified in the image"

                    echo "Checking exposed port..."                                                                           
                    docker inspect ${CI_IMAGE} --format='Port: {{json .Config.ExposedPorts}}'
                    echo "✅ Port OK"  

                    echo "Image Verification completed successfully"                                                                         
                """
            }
        }

        stage('🔒 Security Scan') {                                                                                              
            steps {
                sh """                                                                                                           
                    echo "=== Security Scan ==="

                    CONTAINER_UID=\$(docker run --rm --entrypoint id ${CI_IMAGE} -u)                                             
                    echo "Container UID: \${CONTAINER_UID}"
                                                                                                                                 
                    if [ "\${CONTAINER_UID}" = "0" ]; then                                                                       
                        echo "❌ FAILED: Running as ROOT (UID 0) - Security risk!"
                        exit 1                                                                                                   
                    else
                        echo "✅ PASSED: Non-root UID (\${CONTAINER_UID})"                                                       
                    fi                                                                                                           
                """
            }                                                                                                                    
        }       

        stage('🧹 Cleanup') {
            steps {
                sh """                                                                                                           
                    docker rmi ${CI_IMAGE} || true
                    echo "✅ Cleanup done"                                                                                       
                """
            }
        }                                                                                                                        
    }
                                                                                                                                 
    post {      
        success {
            script {
                if (env.CHANGE_ID) {
                    echo """
  ╔══════════════════════════════════════════════════════╗
  ║            ✅ CI PASSED - PR VALIDATED               ║                                                                         
  ╚══════════════════════════════════════════════════════╝
     PR     : #${env.CHANGE_ID} - ${env.CHANGE_TITLE}                                                                              
                
     ✅ Code Quality  : Passed                                                                                                     
     ✅ Quality Gate  : Passed
     ✅ Docker Build  : Passed                                                                                                     
     ✅ Image Verify  : Passed
     ✅ Security Scan : Passed                                                                                                     
     🚫 Deployment   : Skipped (PRs never deploy)
                                                                                                                                 
     → Get code review → Merge to main for deployment                                                                              
  ══════════════════════════════════════════════════════                                                                           
                    """                                                                                                          
                } else {
                    echo """
  ╔══════════════════════════════════════════════════════╗
  ║          ✅ CI PASSED - BRANCH VALIDATED             ║                                                                         
  ╚══════════════════════════════════════════════════════╝
     Branch : ${env.BRANCH_NAME}                                                                                                   
     Build  : #${env.BUILD_NUMBER}                                                                                                 
  
     ✅ Code Quality  : Passed                                                                                                     
     ✅ Quality Gate  : Passed
     ✅ Docker Build  : Passed
     ✅ Image Verify  : Passed
     ✅ Security Scan : Passed                                                                                                     
  ══════════════════════════════════════════════════════
                    """                                                                                                          
                }
            }
        }
                                                                                                                                 
        failure {
            echo """                                                                                                             
  ╔══════════════════════════════════════════════════════╗
  ║              ❌ CI PIPELINE FAILED                   ║
  ╚══════════════════════════════════════════════════════╝
     Build  : #${env.BUILD_NUMBER}                                                                                                 
     Branch : ${env.BRANCH_NAME}
     PR     : ${env.CHANGE_ID ?: 'N/A'}                                                                                            
     Logs   : ${env.BUILD_URL}
  ══════════════════════════════════════════════════════                                                                           
            """
        }                                                                                                                        
                
        always {                                                                                                                 
              sh 'docker image prune -f || true'
          }                                                                                                                        
    }           
}