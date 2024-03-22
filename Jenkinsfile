pipeline {
  environment {
    ID_DOCKER = "${ID_DOCKER_PARAMS}" // This should be set in Jenkins job parameters
    IMAGE_NAME = "alpinehelloworld"
    IMAGE_TAG = "latest"
    PORT_EXPOSED = "54993" // Host port mapped to the container's exposed port
    STAGING = "${ID_DOCKER}-staging"
    PRODUCTION = "${ID_DOCKER}-production"
  }
  agent none
  stages {
    stage('Build image') {
      agent any
      steps {
        script {
          sh 'docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .'
        }
      }
    }
    stage('Run container based on builded image') {
      agent any
      steps {
        script {
          sh '''
             echo "Clean Environment"
             docker rm -f $IMAGE_NAME || echo "container does not exist"
             docker run --name $IMAGE_NAME -d -p ${PORT_EXPOSED}:5000 -e PORT=5000 ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
             sleep 5
          '''
        }
      }
    }
    stage('Test image') {
      agent any
      steps {
        script {
          sh '''
              curl http://localhost:${PORT_EXPOSED} | grep -q "Hello world!"
          '''
        }
      }
    }
    stage('Clean Container') {
      agent any
      steps {
        script {
          sh '''
            docker stop $IMAGE_NAME
            docker rm $IMAGE_NAME
          '''
        }
      }
    }
    stage('Login and Push Image on docker hub') {
      agent any
      environment {
        DOCKERHUB_PASSWORD = credentials('dockerhub')
      }            
      steps {
        script {
          sh '''
              echo $DOCKERHUB_PASSWORD | docker login -u $ID_DOCKER --password-stdin
              docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
          '''
        }
      }
    }    
    stage('Push image in staging and deploy it') {
      when {
        expression { GIT_BRANCH == 'origin/master' }
      }
      agent any
      environment {
        HEROKU_API_KEY = credentials('heroku_api_key')
      }  
      steps {
        script {
          sh '''
            npm i -g heroku@7.68.0
            heroku container:login
            heroku create $STAGING || echo "project already exist"
            heroku container:push web -a $STAGING
            heroku container:release web -a $STAGING
          '''
        }
      }
    }
    stage('Push image in production and deploy it') {
      when {
        expression { GIT_BRANCH == 'origin/production' }
      }
      agent any
      environment {
        HEROKU_API_KEY = credentials('heroku_api_key')
      }  
      steps {
        script {
          sh '''
            npm i -g heroku@7.68.0
            heroku container:login
            heroku create $PRODUCTION || echo "project already exist"
            heroku container:push web -a $PRODUCTION
            heroku container:release web -a $PRODUCTION
          '''
        }
      }
    }
  }
}
