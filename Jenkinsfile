pipeline {
  environment {
    K8S_NAMESPACE = "gamedragon"
    K8S_DEPLOY_REPLICAS = "${REPLICAS}"
    K8S_RES_REQ_CPU = "1"
    K8S_RES_REQ_RAM = "2G"
    K8S_RES_LIM_CPU = "2"
    K8S_RES_LIM_RAM = "4G"
    QUEUE_CONNECTION = "sync"
    CACHE_DRIVER = "file"
    REDIS_HOST = "10.9.109.15"
    REDIS_PORT = "6379"
    K8S_XMPP_DOMAIN_SECRET_NAME = "chat-gd1.xmpp.oncam.com"
    REGISTRY_HOST = 'uhub.service.ucloud.cn'
    REGISTRY_PATH = 'ws_kubernetes_mirror'
    REGISTRY_AUTH = 'uhub'
    IMAGE_NAME = "ws-api"
    IMAGE_TAG_BASE = "${REGISTRY_HOST}/${REGISTRY_PATH}/${IMAGE_NAME}"
    BRANCH = "${gitlabBranch}"
    LOG_CHANNEL = "${LOG_CHANNEL}"
    LOG_LEVEL = "${LOG_LEVEL}"
    RMQ_HOST = "10.9.151.157"
    RMQ_PORT = "5672"
    RMQ_USER = "oncam"
    RMQ_JOB_QUEUE = "search"
    RMQ_JOB_EXCHANGE = "search"
    RIAK_PORT = "8098"
    RIAK_HOST = "10.9.151.157"
    ELASTIC_HOST = "10.9.97.113"
    JABBER_HOST= "chatweb.oncam.com"
    JABBER_URL = "http://ej:5280/ws_handler"
    DB_PORT = "3307"
    DB_HOST = "10.9.151.157"
    DB_DATABASE = "ws"
    DB_USERNAME = "root"
    UFILE_BUCKET = "${UFILE_BUCKET}"
    APP_ENV = "${APP_ENV}"
    COMPOSER_MIRROR_HOST ="http://10.9.123.23:7777"
    }
  agent {
  label 'docker_slave1'
  }
  stages {
    stage('Clone Git') {
      steps {
        sh 'env'
        dir('api') {
            git branch: "${BRANCH}",
                credentialsId: 'jenkins',
                url: 'git@gitlab.oncam.com:ws/api.git'
            sh "cat composer.json | jq '. + {\"repositories\":[{\"type\":\"composer\",\"url\":\"${COMPOSER_MIRROR_HOST}\"},{\"packagist\":false}]}' | jq '.config.\"secure-http\" = false' > composer_mirror.json"
            sh 'mv composer_mirror.json composer.json && rm composer.lock && cat composer.json'
            script {
                COMMIT_HASH = sh (
                    script: "git rev-parse HEAD | cut -c -7",
                    returnStdout: true
                ).trim()
            }
        }
        dir('ops') {
            git branch: 'separate-indexer',
                credentialsId: 'jenkins',
                url: 'git@gitlab.oncam.com:3d/ops.git'
        }
      }
    }
    stage('Build & push image') {
      steps{
        script {
          env.IMAGE_TAG="${IMAGE_TAG_BASE}" + ":${BRANCH}-${COMMIT_HASH}"
          dir('api') {
              docker.withRegistry( "https://${REGISTRY_HOST}", "${REGISTRY_AUTH}") {
                dockerImage = docker.build("${IMAGE_TAG}", "--pull --no-cache -f Dockerfile_chn .")
                dockerImage.push()
              }
              sh 'docker rmi ${IMAGE_TAG}'
          }
        }
      }
    }
    stage('Deploy') {
      environment {
        JWT_KEY = credentials('DEV_WS_JWT_KEY')
        UFILE_ACCESS_KEY = credentials('DEV_WS_UFILE_ACCESS_KEY')
        UFILE_SECRET_KEY = credentials('DEV_WS_UFILE_SECRET_KEY')
        RMQ_PASS = credentials('DEV_WS_RMQ_PASS')
        DB_PASSWORD = credentials('DEV_WS_DB_PASSWORD')
        K8S_IMAGE = "${IMAGE_TAG}"
        KUBERNETES_IMAGE = "${IMAGE_TAG}"
        REDIS_PASSWORD = credentials('DEV_WS_REDIS_PASSWORD')
        K8S_DEPLOY_VERSION = "${COMMIT_HASH}"
      }
      steps{
        script {
            dir('ops/k8s/templates') {
                kubernetesDeploy(
                    kubeconfigId: 'kube-dev1',
                    configs: 'ws-api.yaml',
                    enableConfigSubstitution: true
                )
                withCredentials([kubeconfigContent(credentialsId: 'kube-dev1', variable: 'KUBE_CONFIG')]) {
                    sh "set +x ; echo \"$KUBE_CONFIG\" > ./config ; set -x"
                    sh 'kubectl --kubeconfig=./config wait -n gamedragon deployment/ws --for condition=available --timeout=2m'
                }
            }
        }
      }
    }
  }
  post { 
    success {
        slackSend color: 'good', message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\nTime taken: ${currentBuild.durationString}\nDeployed image: ${env.IMAGE_TAG}\nChange list: ${env.BUILD_URL}changes"
    }
    failure {
        slackSend color: 'danger', message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}console"
    }
  }
} 
