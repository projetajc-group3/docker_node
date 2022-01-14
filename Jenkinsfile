def EC2_PRODUCTION_HOST = "@IP" 
def EC2_STAGING_HOST = "@IP"

pipeline {
    environment {
        IMAGE_NAME = "node"
        IMAGE_TAG = "1.0"
        USERNAME = "projetajcgroup3"
        CONTAINER_NAME = "webapp"
        CONTAINER_PORT = "3000"
        EXTERNAL_PORT = "30001"
        URL_GIT_NODE ="https://github.com/projetajc-group3/projetajc_node.git"
        URL_GIT_TERRAFORM ="https://github.com/projetajc-group3/terraform_node.git"
        URL_GIT_DEPLOY_DOCKER = "https://github.com/projetajc-group3/docker_role_deploy.git"
        URL_GIT_DEPLOY_KUBERNETES = "https://github.com/projetajc-group3/kubernetes_role_deploy.git"
    }
   
    agent none
  
    stages {
        
        stage ('Image Build (TEST)') {
            agent {label 'Agent-test'}
            steps{
                script{ 
                    sh'''
                    rm -rf docker_node/ || true
                    docker stop $CONTAINER_NAME || true
                    docker rm $CONTAINER_NAME || true
                    docker rmi $USERNAME/$IMAGE_NAME:$IMAGE_TAG || true
                    git clone $URL_GIT_NODE || true
                    cd docker_node
                    docker build -t $USERNAME/$IMAGE_NAME:$IMAGE_TAG .                            
                    '''
                }
            }
        }
        
        stage ('Run build (TEST)') {
            agent {label 'Agent-test'}
            steps{
                script{ 
                    sh'''
                    docker run -d -p $EXTERNAL_PORT:$CONTAINER_PORT --name $CONTAINER_NAME $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }
         
        stage('Scan code (TEST)') {
            agent { label 'Agent-test'}
            steps {
                echo 'Testing...'
                snykSecurity(
                    snykInstallation: 'snyk@latest',
                    snykTokenId: 'snyk-token',
                    targetFile: 'docker_node/package.json',
                    failOnIssues: false,
                )
            }
        }
        
        stage ('Test application (TEST)') {
            agent {label 'Agent-test'}
            steps{
                script{
                   sh '''
                       curl http://localhost:$EXTERNAL_PORT/ | tac | tac | grep -iq "DevOps Foundation"
                   '''
                }
            }
        }
         
        stage ('Clean test environment and save artifact (TEST)') {
           agent {label 'Agent-test'}
           environment{
               PASSWORD = credentials('docker-token')
           }
           steps {
               script{
                   sh '''
                       docker login -u $USERNAME -p $PASSWORD
                       docker push $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                       docker stop $CONTAINER_NAME || true
                       docker rm $CONTAINER_NAME || true
                       docker rmi $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                   '''
               }
           }
       }  
     
       stage ('Creation ec2 instance (STAGING)') {
            agent any
            steps {
                script{
                    sh '''
                    rm -rf src_terraform
                    mkdir src_terraform
                    cd src_terraform
                    git clone $URL_GIT_TERRAFORM
                    cd terraform_node/preprod
                    terraform init -reconfigure
                    terraform apply --auto-approve
                    terraform output ec2_ip > ec2_ip.txt
                    '''
                    env.EC2_STAGING_HOST = sh( script: "sed -e 's/\"//g' src_terraform/terraform_node/preprod/ec2_ip.txt",returnStdout: true).trim()
                    sleep 10
                }
            }
        }
        
        stage ('Deploy application (STAGING)') {
            agent any 
           /* when{
                expression{ GIT_BRANCH == 'origin/master'}
            }*/
            steps{
                withCredentials([sshUserPrivateKey(credentialsId: "ec2_test_private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        script{ 
                            sh'''
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} sudo apt update -y || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} sudo apt install ansible git -y || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} git clone $URL_GIT_DEPLOY_DOCKER || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} ansible-galaxy install -r docker_role_deploy/roles/requirements.yml || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} ansible-playbook -i docker_role_deploy/hosts.yml docker_role_deploy/docker.yml || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} sudo rm -rf docker_role_deploy || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} docker stop $CONTAINER_NAME || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} docker rm $CONTAINER_NAME || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_STAGING_HOST} docker run -d -p $EXTERNAL_PORT:$CONTAINER_PORT --name $CONTAINER_NAME $USERNAME/$IMAGE_NAME:$IMAGE_TAG || true
                            '''
                        }
                    }
                }
            } 
        }
        
        stage ('Creation ec2 instance (PRODUCTION)') {
            agent any
            steps {
                script{
                    timeout(time: 30, unit: "MINUTES") {
                        input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                    }
                    
                    sh '''
                    rm -rf src_terraform
                    mkdir src_terraform
                    cd src_terraform
                    git clone $URL_GIT_TERRAFORM
                    cd terraform_node/prod
                    terraform init -reconfigure
                    terraform apply --auto-approve
                    terraform output ec2_ip > ec2_ip.txt
                    '''
                    env.EC2_PRODUCTION_HOST = sh( script: "sed -e 's/\"//g' src_terraform/terraform_node/prod/ec2_ip.txt",returnStdout: true).trim()
                    sleep 10
                }
            }
        }

        stage ('Deploy application (PRODUCTION)') {
            agent any 
           /* when{
                expression{ GIT_BRANCH == 'origin/master'}
            }*/
            steps{
                withCredentials([sshUserPrivateKey(credentialsId: "ec2_test_private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        script{ 
                            sh'''
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} sudo apt update -y || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} sudo apt install ansible git -y || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} git clone $URL_GIT_DEPLOY_DOCKER || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} ansible-galaxy install -r docker_role_deploy/roles/requirements.yml || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} ansible-playbook -i docker_role_deploy/hosts.yml docker_role_deploy/docker.yml || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} sudo rm -rf docker_role_deploy || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} git clone $URL_GIT_DEPLOY_KUBERNETES || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} ansible-galaxy install -r kubernetes_role_deploy/roles/requirements.yml || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} ansible-playbook -i kubernetes_role_deploy/hosts.yml kubernetes_role_deploy/kubernetes.yml --extra-vars "name_containers=$CONTAINER_NAME image_containers=$USERNAME/$IMAGE_NAME:$IMAGE_TAG" || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} sudo rm -rf kubernetes_role_deploy || true
                            '''
                        }
                    }
                }
            } 
        }
        
        
        
/* 
        stage ('Terraform (Test environment destoy)') {
            agent any
            steps {
                script{
                    sh '''
                    cd src_terraform/terraform_node/test
                    terraform destroy --auto-approve
                    cd ../../..
                    rm -rf src_terraform
                    '''
                }
            }
        }
 */     
   }

    post {
        success{
            slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
        failure {
            slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
    }
}
