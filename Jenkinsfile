pipeline{
    agent any
    
    stages{
        stage("Git Checkout"){
            steps{
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/SurajKhanderao/Mega-Project-CD.git'
            }
        }
        
        stage("kubernetes Deployment"){
            steps{
                withKubeConfig(caCertificate: '', clusterName: 'sk-cluster', contextName: '', credentialsId: 'jenkins-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://1AA3CF98978DFB62611998A28FB527DD.gr7.eu-west-1.eks.amazonaws.com') {
             sh '''
                   set -e

                 kubectl apply -f Manifests/.
                    sleep 30
                    kubectl get pods -n webapps
                    kubectl get service -n webapps
                    kubectl get ing -n webapps
                 '''
               }
            }
        }
    }
}
