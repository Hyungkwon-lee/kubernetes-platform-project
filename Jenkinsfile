pipeline {
  agent any

  tools {
    maven 'Maven3'
    jdk 'JDK21'
  }

  environment {
    IMAGE_NAME = "hklee2748/spring-petclinic:${BUILD_NUMBER}"
  }

  stages {

    stage('STAGE.1 : Checkout') {
      steps {
        git branch: 'main',
            url: 'https://github.com/kwony93/project2.git',
            credentialsId: 'git-creds'
      }
    }

    stage('STAGE.2 : Maven Build') {
      steps {
        dir('spring-petclinic') {
          sh 'mvn clean package -DskipTests'
        }
      }
    }

    stage('STAGE.3 : Docker Build') {
      steps {
        dir('spring-petclinic') {
          sh 'docker build -t $IMAGE_NAME .'
        }
      }
    }

    stage('STAGE.4 : Docker Push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'docker-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
          echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
          docker push $IMAGE_NAME
          docker logout
          '''
        }
      }
    }

    stage('STAGE.5 : Deploy to Kubernetes') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          dir('was') {
            sh '''
            export KUBECONFIG=$KUBECONFIG_FILE
            kubectl set image deployment/user3-was \
            user3-was=$IMAGE_NAME \
            -n user3-was
            kubectl rollout restart deployment user3-was -n user3-was
            '''
          }
        }
      }
    }

  }
}
