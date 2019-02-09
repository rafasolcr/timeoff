pipeline {
  agent none
  parameters {
   string(name: "BRANCH", defaultValue: "master", description: "")
   string(name: "CLOUDFORMATIONCLUSTERNAME", defaultValue: "TOCluster", description: "")
   string(name: "REGION", defaultValue: "us-east-1", description: "")
  }
  environment {
    CLUSTERNAME = ""
    ACCOUNT_ID = "087533072584"
    CREDENTIALS_ID = "2c3c471e-ba61-4f58-bbe3-5348bcc8d95c"

  }
  stages {
    stage ("Docker build and Publish") {
      steps {
        node ("master"){
          checkout([$class: "GitSCM", branches: [[name: "*/${params.BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: "${env.CREDENTIALS_ID}", url: "git@github.com:rafasolcr/${env.JOB_NAME}.git"]]])
          script {
            def customImage = docker.build("${env.JOB_NAME}", ".")
            sh "docker tag ${env.JOB_NAME}:latest ${env.ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${env.JOB_NAME}-${env.BUILD_NUMBER}"
            sh "docker push ${env.ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${env.JOB_NAME}-${env.BUILD_NUMBER}"
            sh "docker tag ${env.ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${env.JOB_NAME}-${env.BUILD_NUMBER} ${env.ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${env.JOB_NAME}-latest"
            sh "docker push ${env.ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${env.JOB_NAME}-latest"
          }
        }
      }
    }
    stage ("Deployment") {
      steps {
        node ("master") {
          script {
            sh """
            cat << EOT > ~/taskdefinition.json
            {
              "family": "${env.JOB_NAME}",
              "containerDefinitions": [
                {
                  "image": "${env.ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/timeoff:${env.BUILD_NUMBER}",
                  "name": "${env.JOB_NAME}",
                  "cpu": 1024,
                  "memory": 985,
                  "essential": true,
                  "portMappings": [
                    {
                      "hostPort": 80,
                      "protocol": "tcp",
                      "containerPort": 3000
                    }
                  ]
                }
              ]
            }
            EOT
            REV=`aws ecs register-task-definition --family ${env.JOB_NAME} --cli-input-json file://taskdefinition.json --region ${params.REGION} --query "taskDefinition.revision" --output=text`
            CLUSTERNAME="${params.CLOUDFORMATIONCLUSTERNAME}"
            CLUSTERARN=`aws ecs list-clusters --output=text --query "clusterArns[?contains(@,'${params.CLUSTERNAME}')]" --region ${params.REGION}
            SERVICEARN=`aws ecs list-services --output=text --cluster $CLUSTERARN --query "serviceArns[?contains(@,'${env.JOB_NAME}')]" --region ${params.REGION}`
            
            aws ecs update-service --cluster $CLUSTERARN --service $SERVICEARN --task-definition ${env.JOB_NAME}:${REV} --region ${params.REGION}`
            """
          }
        }
      }
    }
  }
}