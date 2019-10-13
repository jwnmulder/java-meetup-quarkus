pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    ansiColor('xterm')
  }
  
  environment {
    JAVA_TOOL_OPTIONS="-Djansi.force=true -Duser.home=${WORKSPACE}"
    GIT_SHA_SHORT=sh(script: "git rev-parse --short ${GIT_COMMIT}", returnStdout: true).trim()
    APP_NAME="java-meetup-quarkus"
    APP_IMAGE="jwnmulder/${env.APP_NAME}:1.0-b${env.BUILD_ID}.${env.GIT_SHA_SHORT}"
    KUBE_NAMESPACE="default"
    KUBE_NAME_PREFIX="${env.GIT_BRANCH.minus('origin/').replace('/', '-')}"
    KUBE_DEPLOY_NAME="${env.KUBE_NAME_PREFIX}-${env.APP_NAME}"
  }

  stages {

    stage ('checkout') {
      steps {
        checkout scm
      }
    }

    stage ('build') {
      agent {
        docker {
          image "maven:3.6.2-jdk-11"
          reuseNode true
          args "-e MAVEN_CONFIG=${WORKSPACE}/.m2"
        }
      }
      steps {
        sh "mvn clean package"
        junit '**/target/surefire-reports/*.xml'
      }
    }

    stage ('native build') {
      when { branch 'mastera' }
      agent {
        docker {
          image "quay.io/quarkus/ubi-quarkus-native-image:19.2.0.1"
          reuseNode true
          args "--entrypoint='' --memory=3g"
        }
      }
      steps {
        sh "./mvnw package -Pnative"
        archiveArtifacts 'target/*-runner'
      }
    }

    stage('docker build') {
      steps {
        script {
          def image = docker.build(env.APP_IMAGE, "-f src/main/docker/Dockerfile.jvm --pull .")
          docker.withRegistry("https://registry.hub.docker.com", "docker-hub") {
            image.push()
          }
        }
      }
    }

    stage('prepare deploy') {
      agent {
        docker {
          image 'place1/kube-tools:2019.10.13'
          reuseNode true
          args "--entrypoint=''"
        }
      }
      steps {
        dir('deploy') {
            sh """
              > kustomization.yaml
              kustomize edit add base overlays/jenkins-with-nodeport
              kustomize edit set nameprefix '${env.KUBE_NAME_PREFIX}-'
              kustomize edit add label 'app.kubernetes.io/part-of:${env.KUBE_DEPLOY_NAME}'
              kustomize edit set image '${env.APP_IMAGE}'
              kustomize build > generated.yaml
            """
        }
      }
    }

    stage('deploy') {
      agent {
        docker {
          image 'place1/kube-tools:2019.10.13'
          reuseNode true
          args "--entrypoint=''"
        }
      }
      steps {
        withCredentials([file(credentialsId: "kubectl-config", variable: 'KUBECONFIG')]) {
          sh "kubectl -n ${env.KUBE_NAMESPACE} apply -f deploy/generated.yaml --prune -l 'app.kubernetes.io/part-of=${env.KUBE_DEPLOY_NAME}'"
          sh "kubectl -n ${env.KUBE_NAMESPACE} rollout status  --watch deployment ${env.KUBE_DEPLOY_NAME}"
          sh "kubectl -n ${env.KUBE_NAMESPACE} get deployments,ingress,service -o wide"
          sh "kubectl -n ${env.KUBE_NAMESPACE} logs deployment/${KUBE_DEPLOY_NAME} -c web"

          script {
            def serviceNodePort = sh(script: "kubectl -n ${env.KUBE_NAMESPACE} get service ${env.KUBE_DEPLOY_NAME}-external -o=jsonpath='{.spec.ports[?(@.port==8080)].nodePort}'", returnStdout: true)
            env.APP_TEST_URL = "http://localhost:${serviceNodePort}/"
          }
        }
        addBadge icon: "redo.png", link: "${env.APP_TEST_URL}", text: "Test URL: ${env.APP_TEST_URL}"
      }
    }

    stage('Testing') {
      parallel {
        stage('automated testing') {
          steps {
            sleep 60
          }
        }
        stage('manual approval') {
          steps {
            script {
              sh "echo test"
            }
          }
        }
      }
    }

    stage('cleanup') {
      when { not { branch 'mastera' } }
      agent {
        docker {
          image 'place1/kube-tools:2019.10.13'
          reuseNode true
          args "--entrypoint=''"
        }
      }
      steps {
        withCredentials([file(credentialsId: "kubectl-config", variable: 'KUBECONFIG')]) {
          sh "kubectl -n ${env.KUBE_NAMESPACE} delete -f deploy/generated.yaml"
        }
      }
    }
  }
}
