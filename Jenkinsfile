@Library('dynatrace@master') _

def _VERSION = 'UNKNOWN'
def _TAG = 'UNKNOWN'
def _TAG_DEV = 'UNKNOWN'
def _TAG_STAGING = 'UNKNOWN'


pipeline {
  agent {
    label 'maven'
  }
  environment {
    APP_NAME = "carts"
    ARTEFACT_ID = "sockshop-" + "${env.APP_NAME}"
  }
  stages {
    stage('Maven build') {
      steps {
        // checkout scm
        checkout([$class: 'GitSCM', 
          branches: [[name: "${env.BRANCH_NAME}"]], 
          doGenerateSubmoduleConfigurations: false, 
          extensions: [], 
          userRemoteConfigs: [[url: "https://github.com/${env.GITHUB_ORGANIZATION}/${env.APP_NAME}"]]
        ])
        script {
          _VERSION = readFile('version').trim()
          _TAG = "${env.DOCKER_REGISTRY_URL}:5000/sockshop-registry/${env.ARTEFACT_ID}"
          _TAG_DEV = "${_TAG}:${_VERSION}-${env.BUILD_NUMBER}"
          _TAG_STAGING = "${_TAG}:${_VERSION}"
        }
        container('maven') {
          // we remove the files we dont for this build as we had issues excluding it through maven!
          sh "rm /home/jenkins/workspace/sockshop_carts_master/src/test/java/works/weave/socks/cart/loadtest/*.*"          
          sh 'mvn -B clean package'
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker build -t ${_TAG_DEV} ."
        }
      }
    }
    stage('Docker push to registry'){
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          withCredentials([usernamePassword(credentialsId: 'registry-creds', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
            sh "docker login --username=anything --password=${TOKEN} ${env.DOCKER_REGISTRY_URL}:5000"
            sh "docker tag ${_TAG_DEV} ${_TAG_DEV}"
            sh "docker push ${_TAG_DEV}"
          }
        }
      }
    }
    
    stage('Deploy to dev namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
          createDynatraceDeploymentEvent(
            envId: 'Dynatrace Tenant',
          tagMatchRules: [
              [
              meTypes: [
                  [meType: 'SERVICE']
              ],
              tags: [
                  [context: 'Kubernetes', key: 'app', value: "${env.APP_NAME}"],
                  [context: 'CONTEXTLESS', key: 'environment', value: 'dev']
              ]
              ]
          ]) {
            container('kubectl') {
              sh "sed -i 's#image: .*#image: ${_TAG_DEV}#' manifest/carts.yml"
              sh "sed -i 's#value: to-be-replaced-by-jenkins.*#value: ${_VERSION}-${env.BUILD_NUMBER}#' manifest/carts.yml"
              sh "kubectl -n dev apply -f manifest/carts.yml"
            }
          }
      }
    }
    /*
    stage('Run health check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        echo "Waiting for the service to start..."
        sleep 150

        container('jmeter') {
          script {
            def status = executeJMeter ( 
              scriptName: 'jmeter/basiccheck.jmx', 
              resultsDir: "HealthCheck_${BUILD_NUMBER}",
              serverUrl: "${env.APP_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "HealthCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Health check in dev failed."
            }
          }
        }
      }
    }
    stage('Run functional check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('jmeter') {
          script {
            def status = executeJMeter (
              scriptName: "jmeter/${env.APP_NAME}_load.jmx", 
              resultsDir: "FuncCheck_${BUILD_NUMBER}",
              serverUrl: "${env.APP_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "FuncCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Functional check in dev failed."
            }
          }
        }
      }
    }
    */
    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        container('docker'){
          withCredentials([usernamePassword(credentialsId: 'registry-creds', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
            sh "docker login --username=anything --password=${TOKEN} ${env.DOCKER_REGISTRY_URL}:5000"
            sh "docker tag ${_TAG_DEV} ${_TAG_STAGING}"
            sh "docker push ${_TAG_STAGING}"
          }
        }
      }
    }
    stage('Deploy to staging') {
      when {
        beforeAgent true
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        build job: "k8s-deploy-staging",
          parameters: [
            string(name: 'APP_NAME', value: "${env.APP_NAME}"),
            string(name: 'TAG_STAGING', value: "${_TAG_STAGING}"),
            string(name: 'VERSION', value: "${_VERSION}")
          ]
      }
    }
  }
}
