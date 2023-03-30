library identifier: 'jenkins-core-library@kubernetes/1.0.0', retriever: modernSCM(
  [$class: 'GitSCMSource',
    remote: 'https://github.com/pdrodavi/jenkins-core-library.git'
  ])

pipeline {
    
  agent {
    kubernetes {
      yamlFile 'docker-maven-jnlp-pod-template.yaml'
    }
  }
  
  tools {
    maven "M3"
  }  
  
  stages {

    stage('Checkout') {
      steps {
        container('maven') {
          script {
            sh 'ls -a'
            sh 'pwd'
            def lst = [];

            withCredentials([string(name: 'CREDGH', credentialsId: 'github-rest-token', variable: 'GITHUBRESTJWT')]) {
              
                inputName = input([
                        message: 'Input Name Repository',
                        parameters: [
                                string(name: 'Only Name')
                        ]
                ])

                httpRequest consoleLogResponseBody: true, customHeaders: [[maskValue: false, name: 'Accept', value: 'application/vnd.github+json'], [maskValue: false, name: 'Authorization', value: "Bearer ${GITHUBRESTJWT}"], [maskValue: false, name: 'X-GitHub-Api-Version', value: '2022-11-28']], outputFile: 'branches.json', url: "https://api.github.com/repos/pdrodavi/${inputName}/branches", wrapAsMultipart: false

                def props = readJSON file: "${env.WORKSPACE}/branches.json", returnPojo: true
                props.each { key, value ->
                    lst.add("$key.name")
                }

                inputBranch = input([
                        message: 'Choose Branch',
                        parameters: [
                                choice(name: 'Branches', choices: lst, description: 'Select branch for pipeline')
                        ]
                ])

                //deleteDir()
                println("Repositorio: https://github.com/pdrodavi/${inputName}.git")
                println("Branch selecionada: ${inputBranch}")
                git branch: "${inputBranch}", changelog: false, poll: false, url: 'https://pdrodavi:' + "${GITHUBRESTJWT}" + '@github.com/pdrodavi/' + "${inputName}" + '.git'
                sh 'ls -a'
                sh 'pwd'
            }
          }
        }
      }
    }
      
    stage('Analysis') {
      steps {
          script {
            sh 'ls -a'
            sh 'pwd'
            inputAnalysis = input([
                    message: 'Analysis SonarQube?',
                    parameters: [
                            choice(name: 'Analysis', choices: ['Yes', 'No'], description: 'Run on specific analysis')
                    ]
            ])

            Boolean executeStage = false

            if ("${inputAnalysis}" == 'Yes') {
                executeStage = true
            }

            conditionalStage("Analysis", executeStage) {

                if ("${inputAnalysis}" == 'Yes') {
                    withSonarQubeEnv('sonarqube') {
                        sh "mvn -B clean verify sonar:sonar"
                    }
                    def qualitygate = waitForQualityGate()
                    if (qualitygate.status != "OK") {
                        cleanWs()
                        error "Pipeline aborted due to quality gate failure: ${qualitygate.status}"
                    }
                } else {
                    println("Step Skipped")
                }
            }
          }
      }
    }  

    stage('Package') {
      steps {
          script {
            sh 'ls -a'
            sh 'pwd'
            println("Realizando construção do artefato")
            println("Artifact: " + readMavenPom().getArtifactId())
            println("Version: " + readMavenPom().getVersion())
            sh "mvn -Dmaven.test.skip=true -Dmaven.test.failure.ignore clean package"
          }
      }
    }

    stage('Build Image') {
      steps {
        container('docker') {
          sh 'ls -a'
          sh 'pwd'
          println("Criando a imagem Docker")
          sh "docker build -t pdrodavi/${readMavenPom().getArtifactId()}:latest ."
        }
      }
    }
    
    stage('Publish Image') {
      steps {
          script {
            sh 'ls -a'
            sh 'pwd'
            inputPublish = input([
                    message: 'Publish to Registry?',
                    parameters: [
                            choice(name: 'Publish', choices: ['Yes', 'No'], description: 'Publish image to artifactory')
                    ]
            ])

            Boolean executeStage = false

            if ("${inputPublish}" == 'Yes') {
                executeStage = true
            }

            conditionalStage("Publish Image", executeStage) {
                withDockerRegistry(credentialsId: Constants.JENKINS_JFROG_CREDENTIALS_ID, url: Constants.JENKINS_JFROG_URL_REGISTRY) {
                    sh "docker push pdrodavi/${readMavenPom().getArtifactId()}:latest"
                }
            }
          }
      }
    }

    stage('Deployment') {
      steps {
        container('docker') {
          println("Executando Deploy")

          httpRequest acceptType: 'APPLICATION_JSON', consoleLogResponseBody: true, contentType: 'APPLICATION_JSON', customHeaders: [[maskValue: true, name: 'access-api-key', value: 'F6LAVUvVMKJN1wd6Eq0IN7XNuliwJvE0'], [maskValue: true, name: 'Authorization', value: 'Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkwtVmpvVDlkakVMX1pCQUVReWtwNzIwcXJsTDRXYTduWEJZTlRFdGI2T0EifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJwZWRyb2RhdmktYWRtaW4tc2EtdG9rZW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoicGVkcm9kYXZpLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYTFlNmMzZGEtOTlkYy00MDZhLWI3OTMtYzdiN2JhN2FkZTM2Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOnBlZHJvZGF2aS1hZG1pbiJ9.cdH3CKspk3tDmeRz97gAsHik9wfWCHVjkl3HB4mA36pHRWZtk2Y8mXMqXsAYAYJNsxLavmGMDa616whBiFBvkHyjOwBrovmr4IVYH4vVb8Qyz6m1PksT5dUqKO82iQkg3sgQNo_3dgOp54YQtpHTwa-GCT2baJXPtijPtgMyg5KZ6e25aySRldXC6aTiN03skm7-7p7IrtAdqeSFgCOTtKvDYRykMX3GSgi5eIO5127_Y04TMMPAhRT4zgBupCZLiAyG0d5XzywQIUme6VWUpcKrnkiPoY_TXUGTOLCglNUeoJ2SEGwR08OwUjD_JPOZyhvzESIYeYP87-wgpjlozQ']], httpMode: 'POST', requestBody: '''{
            "metadata": {
                "name": "${readMavenPom().getArtifactId()}-deployment",
                "namespace": "bja-dev"
            },
            "apiVersion": "apps/v1",
            "kind": "Deployment",
            "spec": {
                "template": {
                    "metadata": {
                        "namespace": "bja-dev",
                        "labels": {
                            "app": "${readMavenPom().getArtifactId()}"
                        }
                    },
                    "spec": {
                        "dnsPolicy": "ClusterFirst",
                        "terminationGracePeriodSeconds": 0,
                        "containers": [
                            {
                                "image": "pdrodavi/${readMavenPom().getArtifactId()}:latest",
                                "imagePullPolicy": "IfNotPresent",
                                "name": "${readMavenPom().getArtifactId()}",
                                "ports": [
                                    {
                                        "protocol": "TCP",
                                        "name": "http",
                                        "containerPort": 8080
                                    }
                                ]
                            }
                        ],
                        "restartPolicy": "Always"
                    }
                },
                "replicas": 1,
                "revisionHistoryLimit": 1,
                "selector": {
                    "matchLabels": {
                        "app": "${readMavenPom().getArtifactId()}"
                    }
                }
            }
        }''', responseHandle: 'NONE', url: 'https://gwapi.cwrd.com.br/api/kubernetes/apis/apps/v1/namespaces/bja-dev/deployments', wrapAsMultipart: false

          //httpRequest consoleLogResponseBody: true, outputFile: 'deployment.yaml', url: "https://raw.githubusercontent.com/pdrodavi/pipeline-kubernetes-jenkins/main/deployment.yaml", wrapAsMultipart: false
          sh 'ls -a'
          sh 'pwd'
          //sh 'cat deployment.yaml | sed "s/{{NAME_IMAGE}}/${readMavenPom().getArtifactId()}/g" | kubectl apply -f -'
          //sh "docker build -t pdrodavi/${readMavenPom().getArtifactId()}:latest ."
        }
      }
    }
    
  }
}
