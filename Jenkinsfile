pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  serviceAccountName: jenkins
  containers:
  - name: maven
    image: maven:3.8.6-openjdk-11
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  - name: docker
    image: docker:24.0.5-dind
    command: ["cat"]
    tty: true
    securityContext:
      privileged: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: kubectl
    image: bitnami/kubectl:1.28
    command: ["cat"]
    tty: true
  volumes:
  - name: docker-sock
    hostPath: { path: /var/run/docker.sock, type: Socket }
  - name: maven-cache
    emptyDir: {}
'''
    }
  }

  environment {
    DOCKER_HUB_REPO        = 'bertinmwambuka/spring-petclinic'
    DOCKER_CREDENTIALS_ID  = 'dockerhub-credentials'
    K8S_NAMESPACE          = 'default'
    APP_NAME               = 'spring-petclinic'
    IMAGE_TAG              = "${BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git log -1 --pretty=oneline || true'
      }
    }

    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn -B -q --version && mvn -B clean package -DskipTests && ls -lh target/'
        }
      }
    }

    stage('Test') {
      steps {
        container('maven') {
          sh 'mvn -B test'
        }
      }
      post { always { junit '**/target/surefire-reports/*.xml' } }
    }

    stage('Docker Build') {
      steps {
        container('docker') {
          sh '''
            docker version
            docker build -t ${DOCKER_HUB_REPO}:${IMAGE_TAG} .
            docker tag ${DOCKER_HUB_REPO}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:latest
            docker images | grep spring-petclinic || true
          '''
        }
      }
    }

    stage('Docker Push') {
      steps {
        container('docker') {
          withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
            sh '''
              echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
              docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}
              docker push ${DOCKER_HUB_REPO}:latest
            '''
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''
            kubectl apply -f k8s/deployment.yaml -n ${K8S_NAMESPACE}
            kubectl apply -f k8s/service.yaml -n ${K8S_NAMESPACE}
            kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_HUB_REPO}:${IMAGE_TAG} -n ${K8S_NAMESPACE}
            kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=5m
          '''
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        container('kubectl') {
          sh '''
            kubectl get pods -l app=${APP_NAME} -n ${K8S_NAMESPACE}
            kubectl get svc ${APP_NAME} -n ${K8S_NAMESPACE}
            kubectl wait --for=condition=ready pod -l app=${APP_NAME} -n ${K8S_NAMESPACE} --timeout=300s
          '''
        }
      }
    }
  }

  post {
    success {
      echo "SUCCESS: Deployed ${DOCKER_HUB_REPO}:${IMAGE_TAG} -> http://192.168.1.71:30080"
    }
    always { cleanWs() }
  }
}
