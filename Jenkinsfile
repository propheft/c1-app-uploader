import groovy.json.JsonBuilder

node {
  withEnv(['REPOSITORY=c1-app-uploader']) {
    stage('Pull Image from Git') {
      script {
        git (url: "${scm.userRemoteConfigs[0].url}", credentialsId: "github-auth")
      }
    }
    stage('Build Image') {
      script {
        dbuild = docker.build("${REPOSITORY}:$BUILD_NUMBER")
      }
    }
    stage('Push Image to Registry') {
      script {
        docker.withRegistry("https://${K8S_REGISTRY}", 'ecr:eu-west-1:registry-auth') {
          dbuild.push('latest')
        }
      }
    }
    stage ('Check Image with Trend Micro') {
      withCredentials([
        usernamePassword([
          credentialsId: "aws-ecr",
          usernameVariable: "REGISTRY_USER",
          passwordVariable: "REGISTRY_PASSWORD",
        ])
      ]){
        smartcheckScan([
          imageName: "${K8S_REGISTRY}/${REPOSITORY}:latest",
          smartcheckHost: "${DSSC_SERVICE}",
          smartcheckCredentialsId: "smartcheck-auth",
          insecureSkipTLSVerify: true,
          insecureSkipRegistryTLSVerify: true,
          imagePullAuth: new groovy.json.JsonBuilder([
            aws: [
              region: "eu-west-1",
              accessKeyID: REGISTRY_USER,
              secretAccessKey: REGISTRY_PASSWORD,
              ]
          ]).toString(),
          findingsThreshold: new groovy.json.JsonBuilder([
            malware: 1,
            vulnerabilities: [
              defcon1: 0,
              critical: 50,
              high: 250,
            ],
            contents: [
              defcon1: 0,
              critical: 2,
              high: 0,
            ],
            checklists: [
              defcon1: 0,
              critical: 0,
              high: 20,
            ],
          ]).toString(),
        ])
     }
    stage('Deploy App to Kubernetes') {
      script {
        kubernetesDeploy(configs: "app.yml",
                         kubeconfigId: "kubeconfig",
                         enableConfigSubstitution: true,
                         dockerCredentials: [
                           [credentialsId: "ecr:eu-west-1:registry-auth", url: "${K8S_REGISTRY}"],
                         ])
      }
    }
    stage('DS Scan for Recommendations') {
      withCredentials([string(credentialsId: 'deepsecurity-key', variable: 'DSKEY')]) {
        sh 'curl -X POST https://app.deepsecurity.trendmicro.com/api/scheduledtasks/133 -H "api-secret-key: ${DSKEY}" -H "api-version: v1" -H "Content-Type: application/json" -d "{ \\"runNow\\": \\"true\\" }" '
      }     
    }
  }
}
}
