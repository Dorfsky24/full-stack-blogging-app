// pipeline {
//     agent any
//     tools {
//         jdk "jdk17"
//         maven "maven3"
//     }
//     environment {
//         SCANNER_HOME = tool 'sonar-scanner'
//     }

//     stages {
//         stage('Git Checkout') {
//             steps {
//                 git branch: 'main', url: 'git@github.com:Dorfsky24/full-stack-blogging-app.git'
//             }
//         }
//         stage('Compile') {
//             steps {
//                 sh "mvn compile"
//             }
//         }
//         stage('Trivy FS') {
//             steps {
//                 sh "trivy fs . --format table -o fs.html"
//             }
//         }
//         stage('SonarQube Analysis') {
//             steps {
//                 withSonarQubeEnv('sonarqubeServer') {
//                     sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Blogging-app -Dsonar.projectKey=Blogging-app \
//                           -Dsonar.java.binaries=target'''
//                 }
//             }
//         }
//         stage('Build') {
//             steps {
//                 sh "mvn package"
//             }
//         }
//         stage('Publish Artifacts') {
//             steps {
//                 script {
//                     withMaven(
//                         globalMavenSettingsConfig: 'maven-settings',
//                         maven: 'maven3'
//                     ) {
//                         sh "mvn deploy -DaltDeploymentRepository=nexus::default::http://172.31.24.32:8081/repository/maven-releases/"
//                     }
//                 }
//             }
//         }
//         stage('Docker Build & Tag') {
//             steps {
//                 script {
//                     withDockerRegistry(credentialsId: 'dockerhub-cred', url: 'https://hub.docker.com/repositories/abbeydauda') {
//                         sh "docker build -t abbeydauda/blog-app-project ."
//                     }
//                 }
//             }
//         }
//         stage('Trivy Image Scan') {
//             steps {
//                 sh "trivy image --format table -o image.html abbeydauda/blog-app-project:latest"
//             }
//         }
//         stage('Docker Push Image') {
//             steps {
//                 script {
//                     withDockerRegistry(credentialsId: 'dockerhub-cred', url: 'https://hub.docker.com/repositories/abbeydauda') {
//                         sh "docker push abbeydauda/blog-app-project"
//                     }
//                 }
//             }
//         }
//         stage('K8s Deploy') {
//             steps {
//                 withKubeCredentials(kubectlCredentials: [[
//                     caCertificate: '',
//                     clusterName: 'devopsshack-cluster',
//                     contextName: '',
//                     credentialsId: 'k8s-token',
//                     namespace: 'webapps',
//                     serverUrl: 'https://AD1D9143EC6B3C8A72B36759FA28854D.gr7.eu-west-2.eks.amazonaws.com'
//                 ]]) {
//                     sh "kubectl apply -f deployment-service.yml"
//                     sleep 20
//                 }
//             }
//         }
//         stage('Verify Deployment') {
//             steps {
//                 withKubeCredentials(kubectlCredentials: [[
//                     caCertificate: '',
//                     clusterName: 'devopsshack-cluster',
//                     contextName: '',
//                     credentialsId: 'k8s-token',
//                     namespace: 'webapps',
//                     serverUrl: 'https://AD1D9143EC6B3C8A72B36759FA28854D.gr7.eu-west-2.eks.amazonaws.com'
//                 ]]) {
//                     sh "kubectl get pods"
//                     sh "kubectl get service"
//                 }
//             }
//         }
//     }

      
