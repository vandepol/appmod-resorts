// Jenkinsfile for Liberty App - CI/CD
def templateName = 'resorts-liberty'

openshift.withCluster() {
  env.NAMESPACE = openshift.project()
  env.APP_NAME = "${JOB_NAME}".replaceAll(/-build.*/, '')
  echo "Starting Pipeline for ${APP_NAME}..."
  env.BUILD = "${env.NAMESPACE}"
  env.DEV = "${APP_NAME}-dev"
  env.STAGE = "${APP_NAME}-stage"
  env.PROD = "${APP_NAME}-prod"
  
  env.REGISTRY_ROUTE = "docker-registry.default.svc:5000"
  env.EXTERNAL_IMAGE_REPO_URL = "harbor.jkwong.cloudns.cx"
  env.EXTERNAL_IMAGE_REPO_NAMESPACE = "roland-demo"
  env.EXTERNAL_IMAGE_REPO_CREDENTIALS = "harbor"
  env.DST_IMAGE = "${env.EXTERNAL_IMAGE_REPO_URL}/${env.EXTERNAL_IMAGE_REPO_NAMESPACE}/${env.APP_NAME}:${env.BUILD_NUMBER}"
 
}

pipeline {
  agent {
    label "maven"
  }
  stages {
    stage('preamble') {
        steps {
            script {
                openshift.withCluster() {
                    openshift.withProject() {
                        echo "Using project: ${openshift.project()}"
                        echo "APPLICATION_NAME: ${params.APPLICATION_NAME}"
                    }
                }
            }
        }
    }
    // Build Application using Maven
    stage('Maven build') {
      steps {
        sh """
        env
        mvn -v
        mvn clean package
        """
      }
    }
      
    // Run Maven unit tests
    stage('Unit Test'){
      steps {
        sh """
        mvn -v
        mvn test
        """
      }
    }
      
    // Build Container Image using the artifacts produced in previous stages
    stage('Build Liberty App Image'){
      steps {
        script {
          // Build container image using local Openshift cluster
          openshift.withCluster() {
            openshift.withProject() {
                timeout (time: 10, unit: 'MINUTES') {
               // run the build and wait for completion
                def build = openshift.selector("bc", "${params.APPLICATION_NAME}").startBuild("--from-dir=.")
                                    
                def buildObj = build.object()
                def imageRef = buildObj.status.outputDockerImageReference
                def tmpImg  = imageRef.indexOf("/")
                OUTPUT_IMAGE =  imageRef.substring(tmpImg + 1, imageRef.length())
                // print the build logs
                build.logs('-f')
              }
           }        
          }
        }
      }
    } 
    stage('Promote to Dev') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject() {
              def outputImage = OUTPUT_IMAGE
              println "Tagging image : ${outputImage} as ${env.DEV}/${params.APPLICATION_NAME}:latest"
              openshift.tag("${outputImage}", "${env.DEV}/${params.APPLICATION_NAME}:latest")
            }
          }
        }
      }
    }

    stage('Promote to Stage') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject() {
              def outputImage = OUTPUT_IMAGE
              println "Tagging image : ${outputImage} as ${env.STAGE}/${params.APPLICATION_NAME}:latest"
              openshift.tag("${outputImage}", "${env.STAGE}/${params.APPLICATION_NAME}:latest")
            }
          }
        }
      }
    }

    stage('Promote to Production?') {
      steps {
        script {
          input message: 'Promote ${APPLICATION_NAME} to Production?'
        }
      }
    }
    
    stage ('Push Container Image') {
          agent {
            kubernetes {
              cloud 'openshift'
              label 'skopeo-jenkins'
              yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jkwong/skopeo-jenkins
    tty: true
  serviceAccountName: jenkins
"""
            }
          }

          steps {
            script {
                openshift.withCluster() {
                    openshift.withProject() {
                      
                      def srcImage = env.REGISTRY_ROUTE + "/" + OUTPUT_IMAGE
                     
                      println "source image: ${srcImage}, dest image: ${env.DST_IMAGE}"
                      
                      def openshift_token = readFile "/var/run/secrets/kubernetes.io/serviceaccount/token"
                      echo "Username: AFuser: ${env.AFuser}"
                      echo "Username: AFpasswd: ${env.AFpasswd}"
                      echo "Username: AFuser: ${params.AFuser}"
                      echo "Username: AFpasswd: ${params.AFpasswd}"
                      
                     // withCredentials([usernamePassword(credentialsId: "${env.EXTERNAL_IMAGE_REPO_CREDENTIALS}", passwordVariable: 'username', usernameVariable: 'password')]) {
                              sh """
                              /usr/bin/skopeo copy \
                              --src-creds openshift:${openshift_token} \
                              --src-tls-verify=false \
                              --dest-creds vandepol:42L0LN5we8 \
                              --dest-tls-verify=false \
                              docker://${srcImage} \
                              docker://${env.DST_IMAGE}
                              """
                              println("Image is successfully pushed to https://${env.DST_IMAGE}")
                      //    }
                    }
                }
            }
        }
    }
    stage('Promote to Prod') {
      steps {
        script {
          //  openshift.withCluster('prod-east', 'my-prilvileged-token-id') {
          openshift.withCluster() {
            openshift.withProject() {
              kubectl get pods
              printf "depoy here"
              //openshiftDeploy apiURL: 'https://c1-e.us-east.containers.cloud.ibm.com:20955', authToken: 'vIxFo4nQHncQzgryIXtSrp4r77leA0dBMv89E_f9IRU', depCfg: '', namespace: 'roland-demo-prod-east', verbose: 'false', waitTime: '', waitUnit: 'sec'
            }
          }
        }
      }
    }
  }
}
