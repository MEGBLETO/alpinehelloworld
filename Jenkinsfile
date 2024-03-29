pipeline {
    agent none
    environment {
        ID_DOCKER = "${ID_DOCKER_PARAMS}" // This should be set in Jenkins job parameters
        IMAGE_NAME = "alpinehelloworld"
        IMAGE_TAG = "latest"
        PORT_EXPOSED = "54993" // Host port mapped to the container's exposed port
        STAGING = "${ID_DOCKER}-staging"
        PRODUCTION = "${ID_DOCKER}-production"
        PROD_APP_ENDPOINT = "https://nelzer95-staging-ab629589f27f.herokuapp.com/" // Static production endpoint
        STG_APP_ENDPOINT = "https://nelzer95-staging-ab629589f27f.herokuapp.com/" // Static staging endpoint
    }
    stages {
        stage('Build image') {
            agent any
            steps {
                script {
                    sh '/usr/local/bin/docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .'
                }
            }
        }
        stage('Run container based on built image') {
            agent any
            steps {
                script {
                    sh '''
                       echo "Cleaning Environment"
                       /usr/local/bin/docker rm -f $IMAGE_NAME || echo "Container does not exist"
                       /usr/local/bin/docker run --name $IMAGE_NAME -d -p ${PORT_EXPOSED}:5000 -e PORT=5000 ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
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
                        /usr/local/bin/docker stop $IMAGE_NAME
                        /usr/local/bin/docker rm $IMAGE_NAME
                    '''
                }
            }
        }
        stage('Login and Push Image on Docker Hub') {
            agent any
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-login', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                        sh 'echo $DOCKERHUB_PASS | /usr/local/bin/docker login --username $DOCKERHUB_USER --password-stdin'
                        sh '/usr/local/bin/docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}'
                    }
                }
            }
        }
        stage('Push image to staging and deploy it') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
            agent any
            steps {
                script {
                    withCredentials([string(credentialsId: 'heroku_api_key', variable: 'HEROKU_API_KEY')]) {
                        sh '/usr/local/bin/node /usr/local/bin/heroku container:login'
                        sh '/usr/local/bin/node /usr/local/bin/heroku create $STAGING || echo "Project already exists"'
                        sh '/usr/local/bin/node /usr/local/bin/heroku container:push web -a $STAGING'
                        sh '/usr/local/bin/node /usr/local/bin/heroku container:release web -a $STAGING'
                    }
                }
            }
        }
        stage('Push image to production and deploy it') {
            when {
                expression { GIT_BRANCH == 'origin/production' }
            }
            agent any
            steps {
                script {
                    withCredentials([string(credentialsId: 'heroku_api_key', variable: 'HEROKU_API_KEY')]) {
                        sh '/usr/local/bin/node /usr/local/bin/heroku container:login'
                        sh '/usr/local/bin/node /usr/local/bin/heroku create $PRODUCTION || echo "Project already exists"'
                        sh '/usr/local/bin/node /usr/local/bin/heroku container:push web -a $PRODUCTION'
                        sh '/usr/local/bin/node /usr/local/bin/heroku container:release web -a $PRODUCTION'
                    }
                }
            }
        }
    }
    post {
        success {
            slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL}) - PROD URL => http://${PROD_APP_ENDPOINT} , STAGING URL => http://${STG_APP_ENDPOINT}")
        }
        failure {
            slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }   
    }
}
