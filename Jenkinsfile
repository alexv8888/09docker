pipeline {
    environment {
    registry = ''
    registryCredential = 'dockerhub'
    }
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/alexv8888/09docker'
            }
        }
        stage('Validate') {
            steps {
                      sh 'hadolint Dockerfile | tee -a hadolint_lint.txt'
            }
            post {
                always {
                    archiveArtifacts 'hadolint_lint.txt'
                }
            }
        }
        stage('Build') {
            steps {
               script {
                 dockerImage = docker.build ("alexv8288/hello_fastapi:$BUILD_NUMBER")
                }
            }
        }
        stage('Test') {
            steps {
                script {
                try { 
                sh 'docker run --rm -dp 7777:80 --name hello alexv8288/hello_fastapi:$BUILD_NUMBER'
                sh 'sleep 10'
                sh 'curl -i 127.0.0.1:7777 > curl.txt'
                sh 'grep "HTTP/1.1 200 OK" curl.txt'
                } 
                catch(Exception err){
                  error "Test failed"
                }
                finally {
                sh 'docker container stop hello'
                }
                }
                
            }
        }
        stage('Push') {
            steps {
                script {

                   docker.withRegistry( '', registryCredential ) {
                   dockerImage.push() 
                   dockerImage.push('latest') 
                   
                   }
                }
            }
        }
        stage('Deploy in namespace pre-prod') {

            steps {
                script {
                try{
              withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kubeconffile', namespace: '', serverUrl: '') {
                sh "sed s/latest/$BUILD_NUMBER/ myapp.yaml  > newapp.yaml"     
                sh "kubectl apply -f newapp.yaml --namespace=pre-prod"
                sleep 5
                sh "kubectl get pods --namespace=pre-prod"
              }
                }
               catch(Exception err){
              error "Deployment failed"
            } 
}
            } 
        }
        stage('Deploy in namespace prod') {
           
            steps {
                
                script {
                  catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE'){
                  def depl = true
                  try{
                    input("Pre-prod delpoy done. Deploy in prod?")
                  }
                  catch(err){
                    depl = false
                  }
                  
            try{
              if(depl){
                withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kubeconffile', namespace: '', serverUrl: '') {
                  sh "kubectl apply -f newapp.yaml --namespace=prod"
                  sleep 5
                  sh "kubectl get pods --namespace=prod"
                  sh "kubectl delete -f newapp.yaml --namespace=pre-prod" 
                }
              }
            }
            catch(Exception err){
              error "Deployment failed"
            }
          }
        }
      }
    }  
  }
  post {
    success {
      slackSend (color: '#00FF00', message: "Deployment success: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'")
    }
    failure {
      slackSend (color: '#FF0000', message: "Deployment failed: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'")
    }
  }
} 